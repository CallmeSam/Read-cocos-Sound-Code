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
         struct _hashUpdateEntry *_hashForUpdates;  //对应的是update每帧
         struct _hashSelectorEntry *_hashForTimers;     //对应的是有interval的


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


Scheduler::schedule()
主要分三个类型：
void Scheduler::schedule(const ccSchedulerFunc& callback, void *target, float interval, unsigned int repeat, float delay, bool paused, const std::string& key)
{
    CCASSERT(target, "Argument target must be non-nullptr");
    CCASSERT(!key.empty(), "key should not be empty!");

    tHashTimerEntry *element = nullptr;
    HASH_FIND_PTR(_hashForTimers, &target, element);

    if (! element)
    {
        //calloc(numElements ，sizeOfElement)
        //这种写法不是很懂？？
        element = (tHashTimerEntry *)calloc(sizeof(*element), 1);
        element->target = target;

        HASH_ADD_PTR(_hashForTimers, target, element);

        // Is this the 1st element ? Then set the pause level to all the selectors of this target
        element->paused = paused;
    }
    else
    {
        //？？
        CCASSERT(element->paused == paused, "element's paused should be paused!");
    }

    //该元素中是否有计时器
    //没有则申请10个空间
    
    
    if (element->timers == nullptr)
    {
        element->timers = ccArrayNew(10);
    }
    else 
    {
        //如果有且计时器的key相同，则将interval改变，并跳出
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
        //没有key则多添加一个额外空间
        //当timers容量不够时候，会讲timers的容量扩充到两倍
        ccArrayEnsureExtraCapacity(element->timers, 1);
    }
    //new一个timerTargetCallback对象初始化并添加在该element的timers中
    TimerTargetCallback *timer = new (std::nothrow) TimerTargetCallback();
    timer->initWithCallback(this, callback, target, key, interval, repeat, delay);
    //相当于pushback，同时retain了一下
    ccArrayAppendObject(element->timers, timer);
    timer->release();
}

//和上述基本相同，只是在有关TimerTargetCallback一类之时变为了TimerTargetSelector
void Scheduler::schedule(SEL_SCHEDULE selector, Ref *target, float interval, unsigned int repeat, float delay, bool paused);


bool TimerTargetSelector::initWithSelector(Scheduler* scheduler, SEL_SCHEDULE selector, Ref* target, float seconds, unsigned int repeat, float delay)
{
    _scheduler = scheduler;
    _target = target;
    _selector = selector;
    //基本都是一些参数的赋值，看timer再说
    setupTimerWithInterval(seconds, repeat, delay);
    return true;
}


template <class T>
void scheduleUpdate(T *target, int priority, bool paused)
{
    this->schedulePerFrame([target](float dt){
        target->update(dt);
    }, target, priority, paused);
}

void Scheduler::schedulePerFrame(const ccSchedulerFunc& callback, void *target, int priority, bool paused)
{
    tHashUpdateEntry *hashElement = nullptr;
    HASH_FIND_PTR(_hashForUpdates, &target, hashElement);
    //这一块的逻辑并不是很懂？？
    if (hashElement)
    {
        // check if priority has changed
        //如果优先级改变了
        if ((*hashElement->list)->priority != priority)
        {
            //且上锁状态则不会被标记移除，pause会赋值？？
            if (_updateHashLocked)
            {
                CCLOG("warning: you CANNOT change update priority in scheduled function");
                hashElement->entry->markedForDeletion = false;
                hashElement->entry->paused = paused;
                return;
            }
            else
            {
                // will be added again outside if (hashElement).
                //会被移除，注释说会被再次添加
                unscheduleUpdate(target);
            }
        }
        else
        {
            hashElement->entry->markedForDeletion = false;
            hashElement->entry->paused = paused;
            return;
        }
    }

    // most of the updates are going to be 0, that's way there
    // is an special list for updates with priority 0
    if (priority == 0)
    {
        appendIn(&_updates0List, callback, target, paused);
    }
    else if (priority < 0)
    {
        priorityIn(&_updatesNegList, callback, target, priority, paused);
    }
    else
    {
        // priority > 0
        priorityIn(&_updatesPosList, callback, target, priority, paused);
    }
}

void Scheduler::appendIn(_listEntry **list, const ccSchedulerFunc& callback, void *target, bool paused)
{
    //创建一个tListEntry对象，并根据参数赋值，添加到对应的优先级list中
    tListEntry *listElement = new tListEntry();

    listElement->callback = callback;
    listElement->target = target;
    listElement->paused = paused;
    listElement->priority = 0;
    listElement->markedForDeletion = false;

    DL_APPEND(*list, listElement);

    // update hash entry for quicker access
    //在创建一个tHashUpdateEntry对象并添加到_hashForUpdates中
    tHashUpdateEntry *hashElement = (tHashUpdateEntry *)calloc(sizeof(*hashElement), 1);
    hashElement->target = target;
    hashElement->list = list;
    hashElement->entry = listElement;
    HASH_ADD_PTR(_hashForUpdates, target, hashElement);
}

//只不过不上一个函数多了一个排序逻辑
void Scheduler::priorityIn(tListEntry **list, const ccSchedulerFunc& callback, void *target, int priority, bool paused)
{
    //创建一个tListEntry并进行初始赋值
    tListEntry *listElement = new tListEntry();

    listElement->callback = callback;
    listElement->target = target;
    listElement->priority = priority;
    listElement->paused = paused;
    listElement->next = listElement->prev = nullptr;
    listElement->markedForDeletion = false;

    // empty list ?
    //如果列表若空则直接插入
    if (! *list)
    {
        DL_APPEND(*list, listElement);
    }
    else
    {
        bool added = false;

        //不为空，是否优先级最小，是则pushfront
        for (tListEntry *element = *list; element; element = element->next)
        {
            if (priority < element->priority)
            {
                if (element == *list)
                {
                    DL_PREPEND(*list, listElement);
                }
                else
                {
                    //优先级是否有小于列表中元素的优先级，有则插入
                    listElement->next = element;
                    listElement->prev = element->prev;

                    element->prev->next = listElement;
                    element->prev = listElement;
                }

                added = true;
                break;
            }
        }

        // Not added? priority has the higher value. Append it.
        //优先级是否最大，是则直接pushback
        if (! added)
        {
            DL_APPEND(*list, listElement);
        }
    }

    // update hash entry for quick access
    //同样的最后在放入_hashForUpdates中
    tHashUpdateEntry *hashElement = (tHashUpdateEntry *)calloc(sizeof(*hashElement), 1);
    hashElement->target = target;
    hashElement->list = list;
    hashElement->entry = listElement;
    HASH_ADD_PTR(_hashForUpdates, target, hashElement);
}


//关键的update
// main loop
//dt究竟是什么意思等看了director之后再探讨
void Scheduler::update(float dt)
{
    //？？
    _updateHashLocked = true;

    if (_timeScale != 1.0f)
    {
        dt *= _timeScale;
    }

    //
    // Selector callbacks
    //

    // Iterate over all the Updates' selectors
    tListEntry *entry, *tmp;

    // updates with priority < 0
    DL_FOREACH_SAFE(_updatesNegList, entry, tmp)
    {
        if ((! entry->paused) && (! entry->markedForDeletion))
        {
            entry->callback(dt);
        }
    }

    // updates with priority == 0
    //遍历_
    DL_FOREACH_SAFE(_updates0List, entry, tmp)
    {
        if ((! entry->paused) && (! entry->markedForDeletion))
        {
            entry->callback(dt);
        }
    }

    // updates with priority > 0
    DL_FOREACH_SAFE(_updatesPosList, entry, tmp)
    {
        if ((! entry->paused) && (! entry->markedForDeletion))
        {
            entry->callback(dt);
        }
    }

    // Iterate over all the custom selectors
    for (tHashTimerEntry *elt = _hashForTimers; elt != nullptr; )
    {
        _currentTarget = elt;
        _currentTargetSalvaged = false;

        if (! _currentTarget->paused)
        {
            // The 'timers' array may change while inside this loop
            for (elt->timerIndex = 0; elt->timerIndex < elt->timers->num; ++(elt->timerIndex))
            {
                elt->currentTimer = (Timer*)(elt->timers->arr[elt->timerIndex]);
                elt->currentTimerSalvaged = false;

                elt->currentTimer->update(dt);

                if (elt->currentTimerSalvaged)
                {
                    // The currentTimer told the remove itself. To prevent the timer from
                    // accidentally deallocating itself before finishing its step, we retained
                    // it. Now that step is done, it's safe to release it.
                    elt->currentTimer->release();
                }

                elt->currentTimer = nullptr;
            }
        }

        // elt, at this moment, is still valid
        // so it is safe to ask this here (issue #490)
        elt = (tHashTimerEntry *)elt->hh.next;

        // only delete currentTarget if no actions were scheduled during the cycle (issue #481)
        if (_currentTargetSalvaged && _currentTarget->timers->num == 0)
        {
            removeHashElement(_currentTarget);
        }
    }

    // delete all updates that are marked for deletion
    // updates with priority < 0
    DL_FOREACH_SAFE(_updatesNegList, entry, tmp)
    {
        if (entry->markedForDeletion)
        {
            this->removeUpdateFromHash(entry);
        }
    }

    // updates with priority == 0
    DL_FOREACH_SAFE(_updates0List, entry, tmp)
    {
        if (entry->markedForDeletion)
        {
            this->removeUpdateFromHash(entry);
        }
    }

    // updates with priority > 0
    DL_FOREACH_SAFE(_updatesPosList, entry, tmp)
    {
        if (entry->markedForDeletion)
        {
            this->removeUpdateFromHash(entry);
        }
    }

    _updateHashLocked = false;
    _currentTarget = nullptr;

#if CC_ENABLE_SCRIPT_BINDING
    //
    // Script callbacks
    //

    // Iterate over all the script callbacks
    if (!_scriptHandlerEntries.empty())
    {
        for (auto i = _scriptHandlerEntries.size() - 1; i >= 0; i--)
        {
            SchedulerScriptHandlerEntry* eachEntry = _scriptHandlerEntries.at(i);
            if (eachEntry->isMarkedForDeletion())
            {
                _scriptHandlerEntries.erase(i);
            }
            else if (!eachEntry->isPaused())
            {
                eachEntry->getTimer()->update(dt);
            }
        }
    }
#endif
    //
    // Functions allocated from another thread
    //

    // Testing size is faster than locking / unlocking.
    // And almost never there will be functions scheduled to be called.
    if( !_functionsToPerform.empty() ) {
        _performMutex.lock();
        // fixed #4123: Save the callback functions, they must be invoked after '_performMutex.unlock()', otherwise if new functions are added in callback, it will cause thread deadlock.
        auto temp = _functionsToPerform;
        _functionsToPerform.clear();
        _performMutex.unlock();
        for( const auto &function : temp ) {
            function();
        }
        
    }
}