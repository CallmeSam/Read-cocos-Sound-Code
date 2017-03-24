Scheduler

继承：ref

变量:

//系统的优先级大小,这个是有符号类型最小的值
static const int PRIORITY_SYSTEM;

//用户使用的优先级必须大于等于这个值
const int PRIORITY_NON_SYSTEM_MIN;

//_timescale常规是1.0, 若低于1.0可以理解为慢镜头,反之则是快进
float _timeScale;

//其他相关的变量基本都不懂


结构体：
hashmap相关
uthash


构造函数。
析构函数：
	移除所有计时器

void Scheduler::unscheduleAllWithMinPriority(int minPriority)
{
    // Custom Selectors
    tHashTimerEntry *element = nullptr;
    tHashTimerEntry *nextElement = nullptr;
    for (element = _hashForTimers; element != nullptr;)
    {
        // element may be removed in unscheduleAllSelectorsForTarget
        nextElement = (tHashTimerEntry *)element->hh.next;
        unscheduleAllForTarget(element->target);

        element = nextElement;
    }

    // Updates selectors

    //下面三个的意思差不多，是根据参数来跟
    //对应的三个list进行判断移除_updatesNegList、_updates0List、_updatesPosList
    tListEntry *entry, *tmp;
    if(minPriority < 0)
    {
        //官方注释是移除元素，但是并未看懂
        DL_FOREACH_SAFE(_updatesNegList, entry, tmp)
        {
            if(entry->priority >= minPriority)
            {
                unscheduleUpdate(entry->target);
            }
        }
    }

    if(minPriority <= 0)
    {
        DL_FOREACH_SAFE(_updates0List, entry, tmp)
        {
            unscheduleUpdate(entry->target);
        }
    }

    DL_FOREACH_SAFE(_updatesPosList, entry, tmp)
    {
        if(entry->priority >= minPriority)
        {
            unscheduleUpdate(entry->target);
        }
    }
#if CC_ENABLE_SCRIPT_BINDING
    _scriptHandlerEntries.clear();
#endif
}

void Scheduler::unscheduleAllForTarget(void *target)
{
    // explicit nullptr handling
    if (target == nullptr)
    {
        return;
    }

    // Custom Selectors
    tHashTimerEntry *element = nullptr;
    //在_hashForTimes中需找属于target的element
    HASH_FIND_PTR(_hashForTimers, &target, element);

    if (element)
    {
        //如果element存在的话，在element的timer中是否存在currentTimes，且
        //当前timers没有被回收利用
        if (ccArrayContainsObject(element->timers, element->currentTimer)
            && (! element->currentTimerSalvaged))
        {
        	  //自己理解：当前timer不能直接移除掉，但是要做好标记
            element->currentTimer->retain();
            element->currentTimerSalvaged = true;
        }
        //移除所有的timers
        ccArrayRemoveAllObjects(element->timers);

        //如果当前target是element，也做好回收标记，并不直接移除
        if (_currentTarget == element)
        {
            _currentTargetSalvaged = true;
        }
        else
        {
        	  //若不是当前目标，则直接移除
            removeHashElement(element);
        }
    }

    // update selector
    unscheduleUpdate(target);
}

void Scheduler::removeHashElement(_hashSelectorEntry *element)
{
    //移除element->timers并释放空间
    ccArrayFree(element->timers);
    //在_hashForTimes中移除element并释放。HASH_DEL在utHash中
    HASH_DEL(_hashForTimers, element);
    free(element);
}

void Scheduler::unscheduleUpdate(void *target)
{
    if (target == nullptr)
    {
        return;
    }

    tHashUpdateEntry *element = nullptr;
    HASH_FIND_PTR(_hashForUpdates, &target, element);
    if (element)
    {
        //官方文档注释：如果为true仅仅标记为移除但并不会从hash中移除任何东西
        if (_updateHashLocked)
        {
            element->entry->markedForDeletion = true;
        }
        else
        {
            this->removeUpdateFromHash(element->entry);
        }
    }
}

void Scheduler::removeUpdateFromHash(struct _listEntry *entry)
{
    tHashUpdateEntry *element = nullptr;

    //这个对应的是_hashForUpdates，逻辑上和removeHashElement类似
    HASH_FIND_PTR(_hashForUpdates, &entry->target, element);
    if (element)
    {
        // list entry
        DL_DELETE(*element->list, element->entry);
        CC_SAFE_DELETE(element->entry);

        // hash entry
        HASH_DEL(_hashForUpdates, element);
        free(element);
    }
}

//两个逻辑上基本一样
void unschedule(const std::string& key, void *target);
void unschedule(SEL_SCHEDULE selector, Ref *target)
{
    // explicity handle nil arguments when removing an object
    if (target == nullptr || selector == nullptr)
    {
        return;
    }
    
    //CCASSERT(target);
    //CCASSERT(selector);
    
    //找到该element
    tHashTimerEntry *element = nullptr;
    HASH_FIND_PTR(_hashForTimers, &target, element);
    
    if (element)
    {
        for (int i = 0; i < element->timers->num; ++i)
        {
            TimerTargetSelector *timer = static_cast<TimerTargetSelector*>(element->timers->arr[i]);
            
            //在所有的timer中找到与参数相同的Selector
            if (selector == timer->getSelector())
            {
                //如果是需要回收利用的则标记上
                if (timer == element->currentTimer && (! element->currentTimerSalvaged))
                {
                    element->currentTimer->retain();
                    element->currentTimerSalvaged = true;
                }
                //在timers中移除该timer
                ccArrayRemoveObjectAtIndex(element->timers, i, true);
                
                // update timerIndex in case we are in tick:, looping over the actions
                //英文并不是很懂,理解上应该是如果在进行状态(?)，如果移除的是timer的索引比当前的timer所用下标大
                //需要向前移
                if (element->timerIndex >= i)
                {
                    element->timerIndex--;
                }
                
                //如果已经没有timers，同时没有标记则对其进行移除
                if (element->timers->num == 0)
                {
                    if (_currentTarget == element)
                    {
                        _currentTargetSalvaged = true;
                    }
                    else
                    {
                        removeHashElement(element);
                    }
                }
                
                return;
            }
        }
    }
}


记录schedule的容器分两类：
		 struct _hashUpdateEntry *_hashForUpdates;	//对应的是update每帧
		 struct _hashSelectorEntry *_hashForTimers;  	//对应的是有interval的


struct _listEntry *_updatesNegList;        // list of priority < 0
struct _listEntry *_updates0List;            // list priority == 0
struct _listEntry *_updatesPosList;        // list priority > 0
这三个链表对应的应该是存储了：
	优先级  大于０、等于０、小于０的＿hashUpdateEntry?
	这三个数据应该是和_hashForUpdates同步的

bool _currentTargetSalvaged;
// If true unschedule will not remove anything from a hash. Elements will only be marked for deletion.
bool _updateHashLocked;
这两个标记对应的应是：
		_currentTargetSalvaged ========>  _hashForTimers
		_updateHashLocked        ========>  _hashForUpdates
在不能直接移除时候需要做上标记，等下一时刻移除


Scheduler::update()
主要分三个类型：
void Scheduler::schedule(const ccSchedulerFunc& callback, void *target, float interval, unsigned int repeat, float delay, bool paused, const std::string& key)
{
    CCASSERT(target, "Argument target must be non-nullptr");
    CCASSERT(!key.empty(), "key should not be empty!");

    tHashTimerEntry *element = nullptr;
    HASH_FIND_PTR(_hashForTimers, &target, element);

    if (! element)
    {
        element = (tHashTimerEntry *)calloc(sizeof(*element), 1);
        element->target = target;

        HASH_ADD_PTR(_hashForTimers, target, element);

        // Is this the 1st element ? Then set the pause level to all the selectors of this target
        element->paused = paused;
    }
    else
    {
        CCASSERT(element->paused == paused, "element's paused should be paused!");
    }

    if (element->timers == nullptr)
    {
        element->timers = ccArrayNew(10);
    }
    else 
    {
        for (int i = 0; i < element->timers->num; ++i)
        {
            TimerTargetCallback *timer = dynamic_cast<TimerTargetCallback*>(element->timers->arr[i]);

            if (timer && key == timer->getKey())
            {
                CCLOG("CCScheduler#scheduleSelector. Selector already scheduled. Updating interval from: %.4f to %.4f", timer->getInterval(), interval);
                timer->setInterval(interval);
                return;
            }        
        }
        ccArrayEnsureExtraCapacity(element->timers, 1);
    }

    TimerTargetCallback *timer = new (std::nothrow) TimerTargetCallback();
    timer->initWithCallback(this, callback, target, key, interval, repeat, delay);
    ccArrayAppendObject(element->timers, timer);
    timer->release();
}


void Scheduler::schedule(SEL_SCHEDULE selector, Ref *target, float interval, unsigned int repeat, float delay, bool paused)
{
    CCASSERT(target, "Argument target must be non-nullptr");
    
    tHashTimerEntry *element = nullptr;
    HASH_FIND_PTR(_hashForTimers, &target, element);
    
    if (! element)
    {
        element = (tHashTimerEntry *)calloc(sizeof(*element), 1);
        element->target = target;
        
        HASH_ADD_PTR(_hashForTimers, target, element);
        
        // Is this the 1st element ? Then set the pause level to all the selectors of this target
        element->paused = paused;
    }
    else
    {
        CCASSERT(element->paused == paused, "element's paused should be paused.");
    }
    
    if (element->timers == nullptr)
    {
        element->timers = ccArrayNew(10);
    }
    else
    {
        for (int i = 0; i < element->timers->num; ++i)
        {
            TimerTargetSelector *timer = dynamic_cast<TimerTargetSelector*>(element->timers->arr[i]);
            
            if (timer && selector == timer->getSelector())
            {
                CCLOG("CCScheduler#scheduleSelector. Selector already scheduled. Updating interval from: %.4f to %.4f", timer->getInterval(), interval);
                timer->setInterval(interval);
                return;
            }
        }
        ccArrayEnsureExtraCapacity(element->timers, 1);
    }
    
    TimerTargetSelector *timer = new (std::nothrow) TimerTargetSelector();
    timer->initWithSelector(this, selector, target, interval, repeat, delay);
    ccArrayAppendObject(element->timers, timer);
    timer->release();
}

template <class T>
void scheduleUpdate(T *target, int priority, bool paused)
{
    this->schedulePerFrame([target](float dt){
        target->update(dt);
    }, target, priority, paused);
}



