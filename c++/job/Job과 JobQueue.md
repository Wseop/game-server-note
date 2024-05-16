# Job과 JobQueue
## Command 패턴
**어떤 요청을 객체로 만들어서 나중에 처리할 수 있도록 매개 변수 등의 정보를 저장해두는 패턴**
###
multithread 환경에서 lock을 잡고 처리해야하는 요청이 여러개 발생하는 경우 병목현상이 발생하게 된다.<br>
요청을 바로 처리하지 않고 객체로 만들어서 넘겨준다음 특정 thread에서 단독으로 처리하도록 하면 lock을 잡고 있는 구간이 현저히 줄어들어 문제를 해결할 수 있다.<br>
아래 코드는 multithread 환경에서 GameRoom에 입장과 퇴장하는 함수를 Command 패턴으로 구현한 예제이다.
## GameRoom 구현
- 간단하게 `id`를 받아서 입장, 퇴장 메세지를 출력한다.
- `JobQueue`를 하나 들고 있으며, 함수를 직접 호출하는대신 `JobQueue`에 작업을 넣도록 구현한다.
```cpp
class GameRoom
{
public:
    void enter(int id)
    {
        _ids.insert(id);

        cout << "Enter Room : " << id << endl;
    }

    void exit(int id)
    {
        _ids.erase(id);

        cout << "Exit Room : " << id << endl;
    }

    JobQueue _jobQueue;
    set<int> _ids;
};
```
## JobQueue 구현
- multithread 환경에서 동작할 수 있도록 lock을 사용하였다.
```cpp
class JobQueue
{
public:
    void push(shared_ptr<class IJob> job)
    {
        lock_guard<mutex> lock(_mutex);

        _jobQueue.push(job);
    }

    shared_ptr<class IJob> pop()
    {
        lock_guard<mutex> lock(_mutex);

        if (_jobQueue.empty())
        {
            return nullptr;
        }

        shared_ptr<IJob> job = _jobQueue.front();
        _jobQueue.pop();
        return job;
    }

private:
    mutex _mutex;
    queue<shared_ptr<class IJob>> _jobQueue;
};
```
## Job 구현
### 1. Interface 방식
- `IJob` Interface를 상속받아 각각의 요청에 대한 정보를 기억하는 클래스를 정의.
- 요청 1개당 1개의 클래스를 정의해야하므로 클래스가 매우 많아지게 되는 단점이 있음.
#### IJob Interface
```cpp
class IJob
{
public:
    virtual void execute() abstract;
};
```
#### Enter, Exit Job
- `IJob` Interface를 상속받아 `execute()`함수를 구현한다.
- 인자로 `GameRoom`과 `id` 정보를 전달받아 기억.
```cpp
class EnterJob : public IJob
{
public:
    EnterJob(shared_ptr<GameRoom> room, int id) : _room(room), _id(id)
    {}

    virtual void execute() override
    {
        _room->enter(_id);
    }

private:
    shared_ptr<GameRoom> _room;
    int _id;
};

class ExitJob : public IJob
{
public:
    ExitJob(shared_ptr<GameRoom> room, int id) : _room(room), _id(id)
    {}

    virtual void execute() override
    {
        _room->exit(_id);
    }

private:
    shared_ptr<GameRoom> _room;
    int _id;
};
```
#### main
- 입장과 퇴장을 반복하는 thread를 코어 갯수만큼 생성하여 실행한다.
- `GameRoom`의 `Enter`와 `Exit`를 직접 실행하지 않고 `Job`객체로 만들어 `JobQueue`에 push하는 방식으로 구현해야한다.
- 메인 thread는 `JobQueue`에 들어있는 `Job`을 꺼내서 처리한다.
```cpp
int main()
{
    shared_ptr<GameRoom> gameRoom = make_shared<GameRoom>();

    vector<thread> threads;
    for (int i = 0; i < thread::hardware_concurrency(); i++)
    {
        threads.push_back(thread([gameRoom, i]()
            {
                while (true)
                {
                    gameRoom->_jobQueue.push(make_shared<EnterJob>(gameRoom, i));
                    this_thread::sleep_for(100ms);

                    gameRoom->_jobQueue.push(make_shared<ExitJob>(gameRoom, i));
                    this_thread::sleep_for(100ms);
                }
            }));
    }

    while (true)
    {
        shared_ptr<IJob> job = gameRoom->_jobQueue.pop();
        if (job != nullptr)
        {
            job->execute();
        }
    }
}
```
#### 실행 결과
![image](https://github.com/Wseop/game-server-note/assets/18005580/98218439-0d56-49ad-9c5c-0b8c91033e6a)
#
### 2. template과 람다를 활용하기
- 위의 방식은 각각의 요청마다 클래스를 새로 정의해야한다는 단점이 있다.
- 이를 **variadic template**과 **람다**를 사용하여 해결하는 예제이다.
#### Job 구현
- `JobFunc`에 실행할 람다 함수를 넣어준다.
- 첫번째 생성자는 인자로 람다 함수를 전달해주면 된다.
- 두번째 생성자는 특정 객체의 멤버함수를 `Job`으로 만들고자 할때 사용되며, variadic template으로 parameter를 받아오도록 구현한다.
```cpp
class Job
{
    using JobFunc = function<void(void)>;

public:
    Job(JobFunc jobFunc) : _jobFunc(jobFunc) {}
    
    template<typename T, typename Ret, typename... Args>
    Job(shared_ptr<T> owner, Ret(T::*func)(Args...), Args... args) 
    {
        _jobFunc = [owner, func, args...]()
            {
                (owner.get()->*func)(args...);
            };
    }
    
    void execute()
    {
        _jobFunc();
    }

private:
    JobFunc _jobFunc;
};
```
#### main
- `EnterJob`, `ExitJob`으로 각각 정의하지 않고 인자만 변경하여 원하는 요청을 처리할 수 있게 되었다.
```cpp
int main()
{
    shared_ptr<GameRoom> gameRoom = make_shared<GameRoom>();

    vector<thread> threads;
    for (int i = 0; i < thread::hardware_concurrency(); i++)
    {
        threads.push_back(thread([gameRoom, i]()
            {
                while (true)
                {
                    gameRoom->_jobQueue.push(make_shared<Job>(gameRoom, &GameRoom::enter, i));
                    this_thread::sleep_for(100ms);

                    gameRoom->_jobQueue.push(make_shared<Job>(gameRoom, &GameRoom::exit, i));
                    this_thread::sleep_for(100ms);
                }
            }));
    }

    while (true)
    {
        shared_ptr<Job> job = gameRoom->_jobQueue.pop();
        if (job != nullptr)
        {
            job->execute();
        }
    }
}
```
#### 실행 결과
- 결과 자체는 동일하다. <br>
![image](https://github.com/Wseop/game-server-note/assets/18005580/56a1a97b-336a-4a06-a22c-151789ade40d)
