Timer 

继承：ref
//_elapsed  流逝

void Timer::update(float dt)
{
	//第一次执行只是更新两个数值？？
    if (_elapsed == -1)
    {
        _elapsed = 0;
        _timesExecuted = 0;
        return;
    }

    // accumulate elapsed time
    //加上现在的时间，即是此刻的流逝时间
    _elapsed += dt;
    
    // deal with delay
    //delay的时间如果大于那么_useDelay则为true
    if (_useDelay)
    {
        if (_elapsed < _delay)
        {
            return;
        }
        //应该是执行selector
        trigger(_delay);
        //使用了这个delay，则移除掉
        _elapsed = _elapsed - _delay;
        _timesExecuted += 1;
        _useDelay = false;
        // after delay, the rest time should compare with interval
        //如果到达了规定了次数则终结
        if (!_runForever && _timesExecuted > _repeat)
        {    //unschedule timer
            cancel();
            return;
        }
    }
    
    // if _interval == 0, should trigger once every frame
    //这个比较关键
    //如果_interval小于0，则是一帧(不论是否有卡)执行一次
    //否则如果_interval正常，则是根据此刻流逝的时间与_interval做比较
    //当_elapsed等于0或者小于interval则终止。即卡的话有可能会进行多次计算
    float interval = (_interval > 0) ? _interval : _elapsed;
    while (_elapsed >= interval)
    {
        trigger(interval);
        _elapsed -= interval;
        _timesExecuted += 1;

        if (!_runForever && _timesExecuted > _repeat)
        {
            cancel();
            break;
        }

        if (_elapsed <= 0.f)
        {
            break;
        }
    }
}

//-----------------------------------------------------
TimerTargetSelector
继承：Timer
成员：target、selector初始化为nullptr


/** Initializes a timer with a target, a selector and an interval in seconds, repeat in number of times to repeat, delay in seconds. */
bool TimerTargetSelector::initWithSelector(Scheduler* scheduler, SEL_SCHEDULE selector, Ref* target, float seconds, unsigned int repeat, float delay)
{
    _scheduler = scheduler;
    _target = target;
    _selector = selector;
    //基本都是一些参数的赋值，基类的函数
    setupTimerWithInterval(seconds, repeat, delay);
    return true;
}

//调用回调
void TimerTargetSelector::trigger(float dt)
{
    if (_target && _selector)
    {
        (_target->*_selector)(dt);
    }
}

//移除，这样可以盘清楚之前的逻辑
void TimerTargetSelector::cancel()
{
    _scheduler->unschedule(_selector, _target);
}

//-------------------------------------------------------
TimerTargetCallback
和 TimerTargetSelector类似 \
多了一个std::string _key;