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

-----------------------------------------------
RotateTo
继承：ActionInterval


void RotateTo::calculateAngles(float &startAngle, float &diffAngle, float dstAngle)
{
    //先对初始角度求余
    if (startAngle > 0)
    {
        startAngle = fmodf(startAngle, 360.0f);
    }
    else
    {
        startAngle = fmodf(startAngle, -360.0f);
    }
    //判断向什么方向旋转
    //这里自我感觉cocos处理的不太周全，也应该对两个角度求一下余数
    diffAngle = dstAngle - startAngle;
    if (diffAngle > 180)
    {
        diffAngle -= 360;
    }
    if (diffAngle < -180)
    {
        diffAngle += 360;
    }
}

void RotateTo::startWithTarget(Node *target)
{
    ActionInterval::startWithTarget(target);
    
    if (_is3D)
    {
        _startAngle = _target->getRotation3D();
    }
    else
    {
        _startAngle.x = _target->getRotationSkewX();
        _startAngle.y = _target->getRotationSkewY();
    }

    calculateAngles(_startAngle.x, _diffAngle.x, _dstAngle.x);
    calculateAngles(_startAngle.y, _diffAngle.y, _dstAngle.y);
    calculateAngles(_startAngle.z, _diffAngle.z, _dstAngle.z);
}

void RotateTo::update(float time)
{
    if (_target)
    {
        if(_is3D)
        {
            _target->setRotation3D(Vec3(
                _startAngle.x + _diffAngle.x * time,
                _startAngle.y + _diffAngle.y * time,
                _startAngle.z + _diffAngle.z * time
            ));
        }
        else
        {
#if CC_USE_PHYSICS
            if (_startAngle.x == _startAngle.y && _diffAngle.x == _diffAngle.y)
            {
                _target->setRotation(_startAngle.x + _diffAngle.x * time);
            }
            else
            {
                _target->setRotationSkewX(_startAngle.x + _diffAngle.x * time);
                _target->setRotationSkewY(_startAngle.y + _diffAngle.y * time);
            }
#else
            _target->setRotationSkewX(_startAngle.x + _diffAngle.x * time);
            _target->setRotationSkewY(_startAngle.y + _diffAngle.y * time);
#endif // CC_USE_PHYSICS
        }
    }
}

--------------------------------------------------
RotateBy
继承：ActionInterval
//反向旋转
RotateBy* RotateBy::reverse() const
{
    if(_is3D)
    {
        Vec3 v;
        v.x = - _deltaAngle.x;
        v.y = - _deltaAngle.y;
        v.z = - _deltaAngle.z;
        return RotateBy::create(_duration, v);
    }
    else
    {
        return RotateBy::create(_duration, -_deltaAngle.x, -_deltaAngle.y);
    }
}

--------------------------------------------------
jumpBy
继承：ActionInterval
void JumpBy::update(float t)
{
    // parabolic jump (since v0.8.2)
    if (_target)
    {
        //先求的此次跳跃进行百分比,也即抛物线此时的X坐标点
        float frac = fmodf( t * _jumps, 1.0f );
        //由抛物线公式可得
        //http://superclass.me/archives/83
        float y = _height * 4 * frac * (1 - frac);
        y += _delta.y * t;

        float x = _delta.x * t;
#if CC_ENABLE_STACKABLE_ACTIONS
        Vec2 currentPos = _target->getPosition();

        Vec2 diff = currentPos - _previousPos;
        _startPosition = diff + _startPosition;

        Vec2 newPos = _startPosition + Vec2(x,y);
        _target->setPosition(newPos);

        _previousPos = newPos;
#else
        _target->setPosition(_startPosition + Vec2(x,y));
#endif // !CC_ENABLE_STACKABLE_ACTIONS
    }
}

JumpBy* JumpBy::reverse() const
{
    return JumpBy::create(_duration, Vec2(-_delta.x, -_delta.y),
        _height, _jumps);
}

------------------------------------------------------
jumpTo
继承：jumpBy
jumpTo的pos是destination，而jumpBy的pos是detal

------------------------------------------------------
Bezier
继承：ActionInterval
//下方是3阶贝塞尔曲线公式
//http://www.html-js.com/article/1628
static inline float bezierat( float a, float b, float c, float d, float t )
{
    return (powf(1-t,3) * a + 
            3*t*(powf(1-t,2))*b + 
            3*powf(t,2)*(1-t)*c +
            powf(t,3)*d );
}

-----------------------------------------------------
ScaleTo
继承：ActionInterval
ScaleBy
继承：ScaleTo
ScaleTo是从某一个比例成为另一个比例
ScaleBy是在某个比例的基础上*倍数
-----------------------------------------------------
Blink
继承：ActionInterval

void Blink::stop()
{
    //在动画停止后将该节点恢复到之前的属性(是否可视)
    if(NULL != _target)
        _target->setVisible(_originalState);
    ActionInterval::stop();
}

void Blink::update(float time)
{
    if (_target && ! isDone())
    {
        //一个周期的时间
        float slice = 1.0f / _times;
        //此刻是第几次blink
        float m = fmodf(time, slice);
        //一个周期两个阶段，一个是显示，一个是消失
        _target->setVisible(m > slice / 2 ? true : false);
    }
}

------------------------------------------------------------
FadeTo
继承：ActionInterval
FadeIn
FadeOut
继承：FadeTo
FadeIn是从当前透明度到255
FadeOut是从当前透明度到0

TintBy、TintTo同理
------------------------------------------------------------
DelayTime
继承：ActionInterval
啥都不干
------------------------------------------------------------
ReverseTime
继承：ActionInterval

void ReverseTime::update(float time)
{
    if (_other)
    {
        //动画倒着放
        _other->update(1 - time);
    }
}

Animate
继承：ActionInterval
成员变量
_splitTimes;                        //将每帧播放的时间点存入该集合中
_nextFrame;                         //下一帧
_origFrame;                         //初始帧
_currFrameIndex;                    //当前帧索引
_executedLoops;                     //执行过的循环次数
_animation;                         //执行的动画
_frameDisplayedEvent;               // 
AnimationFrame::DisplayedEventInfo _frameDisplayedEventInfo;    //


bool Animate::initWithAnimation(Animation* animation)
{
    CCASSERT( animation!=nullptr, "Animate: argument Animation must be non-nullptr");

    float singleDuration = animation->getDuration();
    //初始化总时长
    if ( ActionInterval::initWithDuration(singleDuration * animation->getLoops() ) )
    {
        _nextFrame = 0;
        setAnimation(animation);
        _origFrame = nullptr;
        _executedLoops = 0;
        //开辟空间
        _splitTimes->reserve(animation->getFrames().size());

        float accumUnitsOfTime = 0;
        //得到的是perUnitDelay
        float newUnitOfTimeValue = singleDuration / animation->getTotalDelayUnits();

        auto& frames = animation->getFrames();

        for (auto& frame : frames)
        {
            float value = (accumUnitsOfTime * newUnitOfTimeValue) / singleDuration;
            accumUnitsOfTime += frame->getDelayUnits();
            //记录下每帧播的时间点
            _splitTimes->push_back(value);
        }    
        return true;
    }
    return false;
}

void Animate::startWithTarget(Node *target)
{
    ActionInterval::startWithTarget(target);
    Sprite *sprite = static_cast<Sprite*>(target);

    CC_SAFE_RELEASE(_origFrame);

    //如果需要恢复初始帧
    if (_animation->getRestoreOriginalFrame())
    {
        //获取该精灵的图片作为初始帧
        _origFrame = sprite->getSpriteFrame();
        _origFrame->retain();
    }
    _nextFrame = 0;
    _executedLoops = 0;
}

void Animate::update(float t)
{
    // if t==1, ignore. Animation should finish with t==1
    if( t < 1.0f ) {
        t *= _animation->getLoops();

        // new loop?  If so, reset frame counter
        unsigned int loopNumber = (unsigned int)t;
        if( loopNumber > _executedLoops ) {
            _nextFrame = 0;
            _executedLoops++;
        }

        // new t for animations
        t = fmodf(t, 1.0f);
    }

    auto& frames = _animation->getFrames();
    auto numberOfFrames = frames.size();
    SpriteFrame *frameToDisplay = nullptr;

    for( int i=_nextFrame; i < numberOfFrames; i++ ) {
        float splitTime = _splitTimes->at(i);

        if( splitTime <= t ) {
            _currFrameIndex = i;
            AnimationFrame* frame = frames.at(_currFrameIndex);
            frameToDisplay = frame->getSpriteFrame();
            static_cast<Sprite*>(_target)->setSpriteFrame(frameToDisplay);

            const ValueMap& dict = frame->getUserInfo();
            if ( !dict.empty() )
            {
                if (_frameDisplayedEvent == nullptr)
                    _frameDisplayedEvent = new (std::nothrow) EventCustom(AnimationFrameDisplayedNotification);
                
                _frameDisplayedEventInfo.target = _target;
                _frameDisplayedEventInfo.userInfo = &dict;
                _frameDisplayedEvent->setUserData(&_frameDisplayedEventInfo);
                Director::getInstance()->getEventDispatcher()->dispatchEvent(_frameDisplayedEvent);
            }
            _nextFrame = i+1;
        }
        // Issue 1438. Could be more than one frame per tick, due to low frame rate or frame delta < 1/FPS
        else {
            break;
        }
    }
}