Ref

构造函数：
_referenceCount初始化为1 (引用计数)


//引用计数+1
void Ref::retain();

//引用计数-1，如果为0则释放清除
void Ref::release();

//将该目标添加到自动缓冲池中
Ref* Ref::autorelease();

//-------------------------
ActionManager
继承： Ref

//疑问:
//是否在currentAction不为空时候，_currentTarget定是该element
//因currentAction赋值的时候即是在update中，_currentTarget->currentAction = ..
//循环过程都最后都以将currentAction设置为null
void ActionManager::removeAllActionsFromTarget(Node *target)
{
    // explicit null handling
    if (target == nullptr)
    {
        return;
    }

    tHashElement *element = nullptr;
    HASH_FIND_PTR(_targets, &target, element);
    if (element)
    {
    	//该element中是否存在该action，同时并没有被标记移除，则retain一下标记移除
        if (ccArrayContainsObject(element->actions, element->currentAction) && (! element->currentActionSalvaged))
        {
            element->currentAction->retain();
            element->currentActionSalvaged = true;
        }
        //所有action release()

        ccArrayRemoveAllObjects(element->actions);
        //如果当前target是element，则标记移除，否则直接移除element
        if (_currentTarget == element)
        {
            _currentTargetSalvaged = true;
        }
        else
        {
        	//见scheduler
            deleteHashElement(element);
        }
    }
    else
    {
//        CCLOG("cocos2d: removeAllActionsFromTarget: Target not found");
    }
}

void ActionManager::addAction(Action *action, Node *target, bool paused)
{
    CCASSERT(action != nullptr, "action can't be nullptr!");
    CCASSERT(target != nullptr, "target can't be nullptr!");

    tHashElement *element = nullptr;
    // we should convert it to Ref*, because we save it as Ref*
    Ref *tmp = target;
    HASH_FIND_PTR(_targets, &tmp, element);
    if (! element)
    {
    	//如果element不存在
        element = (tHashElement*)calloc(sizeof(*element), 1);
        element->paused = paused;
        //schedler为什么没有retain？？
        target->retain();
        element->target = target;
        HASH_ADD_PTR(_targets, target, element);
    }

    //分配空间，如果是空，则申请，如果已满则拓宽两倍
     actionAllocWithHashElement(element);
 
     CCASSERT(! ccArrayContainsObject(element->actions, action), "action already be added!");
     //将action添加到element的actions(array)中
     ccArrayAppendObject(element->actions, action);
 	 //需要去看action
     action->startWithTarget(target);
}



// main loop
void ActionManager::update(float dt)
{
    for (tHashElement *elt = _targets; elt != nullptr; )
    {
        _currentTarget = elt;
        _currentTargetSalvaged = false;

        if (! _currentTarget->paused)
        {
            // The 'actions' MutableArray may change while inside this loop.
            for (_currentTarget->actionIndex = 0; _currentTarget->actionIndex < _currentTarget->actions->num;
                _currentTarget->actionIndex++)
            {
                _currentTarget->currentAction = (Action*)_currentTarget->actions->arr[_currentTarget->actionIndex];
                if (_currentTarget->currentAction == nullptr)
                {
                    continue;
                }

                _currentTarget->currentActionSalvaged = false;

                _currentTarget->currentAction->step(dt);

                if (_currentTarget->currentActionSalvaged)
                {
                    // The currentAction told the node to remove it. To prevent the action from
                    // accidentally deallocating itself before finishing its step, we retained
                    // it. Now that step is done, it's safe to release it.
                    //这个被标记删除的action已经在array中移除，只能在currentAction中找到
                    //这个release是真的被移除
                    _currentTarget->currentAction->release();
                } else
                if (_currentTarget->currentAction->isDone())
                {
                    _currentTarget->currentAction->stop();

                    Action *action = _currentTarget->currentAction;
                    // Make currentAction nil to prevent removeAction from salvaging it.
                    _currentTarget->currentAction = nullptr;
                    removeAction(action);
                }

                _currentTarget->currentAction = nullptr;
            }
        }

        // elt, at this moment, is still valid
        // so it is safe to ask this here (issue #490)
        elt = (tHashElement*)(elt->hh.next);

        // only delete currentTarget if no actions were scheduled during the cycle (issue #481)
        if (_currentTargetSalvaged && _currentTarget->actions->num == 0)
        {
            deleteHashElement(_currentTarget);
        }
    }

    // issue #635
    _currentTarget = nullptr;
}

//其他的函数和scheduler不尽相同
//Action


