Action
继承：ref 和 clonable
只是一个框架

主要参数：
Node    *_originalTarget;
Node    *_target;
主要函数：
//ActionManager里面调用的是step
virtual void step(float dt);
/** 
 * Called once per frame. time a value between 0 and 1.

 * For example:
 * - 0 Means that the action just started.
 * - 0.5 Means that the action is in the middle.
 * - 1 Means that the action is over.
 *
 * @param time A value between 0 and 1.
 */
virtual void update(float time);
virtual void startWithTarget(Node *target);

-------------------------------------------

FiniteTimeAction
继承： Action
多了一个_duration

-------------------------------------------

ActionInterval
继承：FiniteTimeAction

//根据duration来初始化action
bool ActionInterval::initWithDuration(float d)
{
    _duration = d;

    // prevent division by 0
    // This comparison could be in step:, but it might decrease the performance
    // by 3% in heavy based action games.
    //在step中会进行除法，为了防止_duration为0则改变其值
    if (_duration == 0)
    {
        _duration = FLT_EPSILON;
    }

    _elapsed = 0;
    _firstTick = true;

    return true;
}

//流逝的时间如果大于等于duration则是完成
bool ActionInterval::isDone() const
{
    return _elapsed >= _duration;
}

//firstTick如果为true，则this->update(0)
//对应上面Action
void ActionInterval::step(float dt)
{
    if (_firstTick)
    {
        _firstTick = false;
        _elapsed = 0;
    }
    else
    {
        _elapsed += dt;
    }
    
    
    float updateDt = MAX (0,                                  // needed for rewind. elapsed could be negative
                           MIN(1, _elapsed /
                               MAX(_duration, FLT_EPSILON)   // division by 0
                               )
                           );

    if (sendUpdateEventToScript(updateDt, this)) return;
    //action比率
    this->update(updateDt);
}



-------------------------------------------
Speed
继承：Action
类成员：
//播放速率，1为正常
float _speed;
//需要改变速率的action
ActionInterval *_innerAction;


Speed* Speed::create(ActionInterval* action, float speed)
{
    Speed *ret = new (std::nothrow) Speed();
    if (ret && ret->initWithAction(action, speed))
    {
    	//丢进缓冲池
        ret->autorelease();
        return ret;
    }
    CC_SAFE_DELETE(ret);
    return nullptr;
}


bool Speed::initWithAction(ActionInterval *action, float speed)
{
    CCASSERT(action != nullptr, "action must not be NULL");
    //retain对应析构函数_innerAction的release
    action->retain();
    _innerAction = action;
    _speed = speed;
    return true;
}

Speed *Speed::clone() const
{
    // no copy constructor
    auto a = new (std::nothrow) Speed();
    //_innerAction克隆
    a->initWithAction(_innerAction->clone(), _speed);
    a->autorelease();
    return a;
}

void Speed::step(float dt)
{
    _innerAction->step(dt * _speed);
}

---------------------------------------------
Follow
继承：Action
//follow
//bg->runAction(Follow::create(node));
//即bg跟着node移动，感觉作用不大
Follow::~Follow()
{
    CC_SAFE_RELEASE(_followedNode);
}

Follow* Follow::create(Node *followedNode, const Rect& rect/* = Rect::ZERO*/)
{
    Follow *follow = new (std::nothrow) Follow();
    if (follow && follow->initWithTarget(followedNode, rect))
    {
        follow->autorelease();
        return follow;
    }
    CC_SAFE_DELETE(follow);
    return nullptr;
}

Follow* Follow::clone() const
{
    // no copy constructor
    auto a = new (std::nothrow) Follow();
    a->initWithTarget(_followedNode, _worldRect);
    a->autorelease();
    return a;
}

Follow* Follow::reverse() const
{
    return clone();
}

bool Follow::initWithTarget(Node *followedNode, const Rect& rect/* = Rect::ZERO*/)
{
    CCASSERT(followedNode != nullptr, "FollowedNode can't be NULL");
 
    followedNode->retain();
    _followedNode = followedNode;
    _worldRect = rect;
    //是否设置了边界
    _boundarySet = !rect.equals(Rect::ZERO);
    _boundaryFullyCovered = false;

    Size winSize = Director::getInstance()->getWinSize();
    _fullScreenSize.set(winSize.width, winSize.height);
    _halfScreenSize = _fullScreenSize * 0.5f;

    //看给的边界是否大于fullScreenSize，如果宽和高都小，则标记上
    //就不需要迭代了
    if (_boundarySet)
    {
        _leftBoundary = -((rect.origin.x+rect.size.width) - _fullScreenSize.x);
        _rightBoundary = -rect.origin.x ;
        _topBoundary = -rect.origin.y;
        _bottomBoundary = -((rect.origin.y+rect.size.height) - _fullScreenSize.y);

        if(_rightBoundary < _leftBoundary)
        {
            // screen width is larger than world's boundary width
            //set both in the middle of the world
            _rightBoundary = _leftBoundary = (_leftBoundary + _rightBoundary) / 2;
        }
        if(_topBoundary < _bottomBoundary)
        {
            // screen width is larger than world's boundary width
            //set both in the middle of the world
            _topBoundary = _bottomBoundary = (_topBoundary + _bottomBoundary) / 2;
        }

        if( (_topBoundary == _bottomBoundary) && (_leftBoundary == _rightBoundary) )
        {
            _boundaryFullyCovered = true;
        }
    }
    
    return true;
}

void Follow::step(float dt)
{
    CC_UNUSED_PARAM(dt);

    if(_boundarySet)
    {
        // whole map fits inside a single screen, no need to modify the position - unless map boundaries are increased
        if(_boundaryFullyCovered)
        {
            return;
        }
        //是否到达边界，到达边界则就依照边界值来设置，否则按照带路人的位置来改变当前位置
        Vec2 tempPos = _halfScreenSize - _followedNode->getPosition();

        _target->setPosition(clampf(tempPos.x, _leftBoundary, _rightBoundary),
                                   clampf(tempPos.y, _bottomBoundary, _topBoundary));
    }
    else
    {
        _target->setPosition(_halfScreenSize - _followedNode->getPosition());
    }
}


----------------------------------------------------------------
Sequence
//Sequence
//sequence实际上是把N个action分解成一个action和另一个action合成一个seq再和第三个action合成seq..

//va_list, va_start,va_arg,va_end是C语言中的知识，主要是处理不定参数
Sequence* Sequence::create(FiniteTimeAction *action1, ...)
{
    va_list params;
    //定位第一个可选参数
    va_start(params, action1);
    
    Sequence *ret = Sequence::createWithVariableList(action1, params);
    //将指针置为空
    va_end(params);
    
    return ret;
}

Sequence* Sequence::createWithVariableList(FiniteTimeAction *action1, va_list args)
{
    FiniteTimeAction *now;
    FiniteTimeAction *prev = action1;
    bool bOneAction = true;

    while (action1)
    {
        //从args中获得下一个类型为FiniteTimeAction的参数
        now = va_arg(args, FiniteTimeAction*);
        if (now)
        {
            //创建为Sequence
            prev = createWithTwoActions(prev, now);
            bOneAction = false;
        }
        else
        {
            // If only one action is added to Sequence, make up a Sequence by adding a simplest finite time action.
            if (bOneAction)
            {
                //ExtraAction如上所说，只是为了处理类似Seq、Spawn这种action中只有一个可执行的action
                prev = createWithTwoActions(prev, ExtraAction::create());
            }
            break;
        }
    }
    
    return ((Sequence*)prev);
}

void Sequence::startWithTarget(Node *target)
{
    ActionInterval::startWithTarget(target);
    //得到第一个action占所有时间的比重
    _split = _actions[0]->getDuration() / _duration;
    //指代的应是当前action播放到第几个，-1应是还未开始播放
    _last = -1;
}

//t为当前Seq运行的进度
void Sequence::update(float t)
{
    //此刻应该执行第一个action
    int found = 0;
    float new_t = 0.0f;

    //判断是否当前进度已经超过了第一个Action
    //接下来判断当前指定action应该播放的进度.
    if( t < _split ) {
        // action[0]
        found = 0;
        if( _split != 0 )
            new_t = t / _split;
        else
            new_t = 1;

    } else {
        // action[1]
        found = 1;
        if ( _split == 1 )
            new_t = 1;
        else
            new_t = (t-_split) / (1 - _split );
    }

    if ( found==1 ) {
        //判断若此刻应该执行第二个action的时候，第一个action未执行完，或者被跳过
        if( _last == -1 ) {
            // action[0] was skipped, execute it.
            _actions[0]->startWithTarget(_target);
            _actions[0]->update(1.0f);
            _actions[0]->stop();
        }
        else if( _last == 0 )
        {
            // switching to action 1. stop action 0.
            _actions[0]->update(1.0f);
            _actions[0]->stop();
        }
    }
    //未懂，应该针对reverse的情况
    else if(found==0 && _last==1 )
    {
        // Reverse mode ?
        // FIXME: Bug. this case doesn't contemplate when _last==-1, found=0 and in "reverse mode"
        // since it will require a hack to know if an action is on reverse mode or not.
        // "step" should be overriden, and the "reverseMode" value propagated to inner Sequences.
        _actions[1]->update(0);
        _actions[1]->stop();
    }
    // Last action found and it is done.
    if( found == _last && _actions[found]->isDone() )
    {
        return;
    }

    // Last action found and it is done
    if( found != _last )
    {
        _actions[found]->startWithTarget(_target);
    }

    _actions[found]->update(new_t);
    _last = found;
}


Sequence* Sequence::reverse() const
{
    //两个action顺序掉转，同时也reverse
    return Sequence::createWithTwoActions(_actions[1]->reverse(), _actions[0]->reverse());
}

------------------------------------------------
Repeat
继承：ActionInterval
bool Repeat::initWithAction(FiniteTimeAction *action, unsigned int times)
{
    float d = action->getDuration() * times;

    if (ActionInterval::initWithDuration(d))
    {
        //重复次数
        _times = times;
        //需要执行的action
        _innerAction = action;
        action->retain();

        _actionInstant = dynamic_cast<ActionInstant*>(action) ? true : false;
        //an instant action needs to be executed one time less in the update method since it uses startWithTarget to execute the action
        if (_actionInstant) 
        {
            _times -=1;
        }
        _total = 0;

        return true;
    }

    return false;
}


void Repeat::update(float dt)
{
    if (dt >= _nextDt)
    {
        while (dt > _nextDt && _total < _times)
        {
            //如果当前的进度超过了当前阶段的一个Action时长
            //则直接执行完此Action
            _innerAction->update(1.0f);
            //记录次数
            _total++;

            //初始化
            _innerAction->stop();
            _innerAction->startWithTarget(_target);
            //更新下一个_nextDt
            _nextDt = _innerAction->getDuration()/_duration * (_total+1);
        }

        // fix for issue #1288, incorrect end value of repeat
        if(dt >= 1.0f && _total < _times) 
        {
            _total++;
        }

        // don't set an instant action back or update it, it has no use because it has no duration
        if (!_actionInstant)
        {
            //如果次数达到了则完成改Action，并清空
            if (_total == _times)
            {
                _innerAction->update(1);
                _innerAction->stop();
            }
            else
            {
                // issue #390 prevent jerk, use right update
                _innerAction->update(dt - (_nextDt - _innerAction->getDuration()/_duration));
            }
        }
    }
    else
    {
        //正常迭代该action
        _innerAction->update(fmodf(dt * _times,1.0f));
    }
}

--------------------------------------------------
RepeatForevet
继承：ActionInterval

void RepeatForever::step(float dt)
{
    //对应的action进行迭代
    _innerAction->step(dt);
    if (_innerAction->isDone())
    {
        float diff = _innerAction->getElapsed() - _innerAction->getDuration();
        if (diff > _innerAction->getDuration())
            //求浮点数的余数
            diff = fmodf(diff, _innerAction->getDuration());
        _innerAction->startWithTarget(_target);
        // to prevent jerk. issue #390, 1247
        _innerAction->step(0.0f);
        _innerAction->step(diff);
    }
}

bool RepeatForever::isDone() const
{
    return false;
}

---------------------------------------------------
Spawn
继承：ActionInterval

//两个action同时开始播放
//一个播放完了进入延时状态
bool Spawn::initWithTwoActions(FiniteTimeAction *action1, FiniteTimeAction *action2)
{
    CCASSERT(action1 != nullptr, "action1 can't be nullptr!");
    CCASSERT(action2 != nullptr, "action2 can't be nullptr!");

    bool ret = false;

    float d1 = action1->getDuration();
    float d2 = action2->getDuration();

    if (ActionInterval::initWithDuration(MAX(d1, d2)))
    {
        _one = action1;
        _two = action2;

        if (d1 > d2)
        {
            _two = Sequence::createWithTwoActions(action2, DelayTime::create(d1 - d2));
        } 
        else if (d1 < d2)
        {
            _one = Sequence::createWithTwoActions(action1, DelayTime::create(d2 - d1));
        }

        _one->retain();
        _two->retain();

        ret = true;
    }

    return ret;
}
