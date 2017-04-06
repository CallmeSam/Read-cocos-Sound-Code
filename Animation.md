AnimationFrame
继承：Ref，Clonable
成员变量：
_spriteFrame                    //当前精灵帧
_delayUnits                     //大抵是，当前帧存在的时间比例(1是对应 1 * animation中的perUnitDelay)
ValueMap _userInfo;             //AnimationFrameDisplayerNotification会在Frame和userInfo显示的时候检测，如果userInfo为空，则不会进行发布消息。

bool AnimationFrame::initWithSpriteFrame(SpriteFrame* spriteFrame, float delayUnits, const ValueMap& userInfo)
{
    setSpriteFrame(spriteFrame);
    setDelayUnits(delayUnits);
    setUserInfo(userInfo);

    return true;
}




Animation
继承：Ref, clonable
成员变量：
_loops                          //循环次数
_delayPerUnit                   //每一帧延迟时间
_duration                       //动画总时长
_restoreOriginalFrame           //是否在动画播放完后恢复到原先帧
_totalDelayUnits                //总延时单元数
_frames                         //动画帧集合

bool Animation::initWithSpriteFrames(const Vector<SpriteFrame*>& frames, float delay/* = 0.0f*/, unsigned int loops/* = 1*/)
{
    _delayPerUnit = delay;
    _loops = loops;

    for (auto& spriteFrame : frames)
    {
        //根据SpriteFrame来创建AnimationFrame
        auto animFrame = AnimationFrame::create(spriteFrame, 1, ValueMap());
        _frames.pushBack(animFrame);
        //增加延时单元
        _totalDelayUnits++;
    }

    return true;
}


float Animation::getDuration(void) const
{
    //总时间是 总延时单元数 * ，每个单元延时时间
    return _totalDelayUnits * _delayPerUnit;
}