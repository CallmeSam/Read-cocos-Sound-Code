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


//Sequence


