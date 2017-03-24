# Sprite

## 继承：Node TextureProtocol

### 构造函数：

CC_SPRITE_DEBUG_DRAW    //宏



sprite的create()函数和init()函数

最终会调用到

bool Sprite::initWithTexture(Texture2D *texture, const Rect& rect, bool rotated)

{

    bool result;

    if (Node::init())

    {

        _batchNode = nullptr;

        //是否所有精灵的孩子需要更新

        _recursiveDirty = false;

        //_dirty = false, 是否这个精灵需要更新

        setDirty(false);



        // opacity and RGB protocol

        _opacityModifyRGB = true;



        //BlendFunc::ALPHA_PREMULTIPLIED = {GL_ONE, GL_ONE_MINUS_SRC_ALPHA};

        //源文件所有都取，目标文件只取源文件中1-alpha值的

        _blendFunc = BlendFunc::ALPHA_PREMULTIPLIED;

        

        _flippedX = _flippedY = false;

        

        // default transform anchor: center

        setAnchorPoint(Vec2(0.5f, 0.5f));

        

        //偏移位置设置为0，偏移位置还未搞懂

        _offsetPosition.setZero();



        // clean the Quad,如前所说是清理quad

        memset(&_quad, 0, sizeof(_quad));

        

        // Atlas: Color  颜色集

        _quad.bl.colors = Color4B::WHITE;

        _quad.br.colors = Color4B::WHITE;

        _quad.tl.colors = Color4B::WHITE;

        _quad.tr.colors = Color4B::WHITE;

        

        // shader state

        setGLProgramState(GLProgramState::getOrCreateWithGLProgramName(GLProgram::SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP));



        // update texture (calls updateBlendFunc)  转下方函数

        setTexture(texture);

        setTextureRect(rect, rotated, rect.size);

        

        // by default use "Self Render".

        // if the sprite is added to a batchnode, then it will automatically switch to "batchnode Render"

        //？？？

        //和setTextureRect有重复代码部分

        setBatchNode(nullptr);

        result = true;

    }

    else

    {

        result = false;

    }

    _recursiveDirty = true;

    setDirty(true);

    return result;

}



//有重载，最终调用的是这个

void Sprite::setTexture(Texture2D *texture)

{

    // If batchnode, then texture id should be the same

    //batchNode需要的texture需要是一样的

    CCASSERT(! _batchNode || (texture &&  texture->getName() == _batchNode->getTexture()->getName()), "CCSprite: Batched sprites should use the same texture as the batchnode");

    // accept texture==nil as argument

    CCASSERT( !texture || dynamic_cast<Texture2D*>(texture), "setTexture expects a Texture2D. Invalid argument");



    if (texture == nullptr)

    {

        // Gets the texture by key firstly.

        //这个是一个默认的材质名字，用于在创建空的精灵时候创建的，为了保证opacity和color正确运行

        //可以见该文档cc_2x2_white_image的注释

        texture = Director::getInstance()->getTextureCache()->getTextureForKey(CC_2x2_WHITE_IMAGE_KEY);



        // If texture wasn't in cache, create it from RAW data.

        //有关image的可以在看之后的文档在回头看！！！！

        if (texture == nullptr)

        {

            Image* image = new (std::nothrow) Image();

            bool isOK = image->initWithRawData(cc_2x2_white_image, sizeof(cc_2x2_white_image), 2, 2, 8);

            CC_UNUSED_PARAM(isOK);

            CCASSERT(isOK, "The 2x2 empty texture was created unsuccessfully.");



            texture = Director::getInstance()->getTextureCache()->addImage(image, CC_2x2_WHITE_IMAGE_KEY);

            CC_SAFE_RELEASE(image);

        }

    }



    //切换素材

    if (!_batchNode && _texture != texture)

    {

        CC_SAFE_RETAIN(texture);

        CC_SAFE_RELEASE(_texture);

        _texture = texture;

        //更新blendFunc，见下

        updateBlendFunc();

    }

}



void Sprite::updateBlendFunc(void)

{

    CCASSERT(! _batchNode, "CCSprite: updateBlendFunc doesn't work when the sprite is rendered using a SpriteBatchNode");



    // it is possible to have an untextured spritec

    //？？

    //是否通过不透明度来改变RGB值并不是很懂

    if (! _texture || ! _texture->hasPremultipliedAlpha())

    {

        _blendFunc = BlendFunc::ALPHA_NON_PREMULTIPLIED;

        setOpacityModifyRGB(false);

    }

    else

    {

        _blendFunc = BlendFunc::ALPHA_PREMULTIPLIED;

        setOpacityModifyRGB(true);

    }

}



//是否和当前设置冲突，如果冲突则更新颜色

void Sprite::setOpacityModifyRGB(bool modify);



//前半部本即是否根据透明度改变值,后半部分不懂

void Sprite::updateColor(void);





void Sprite::setTextureRect(const Rect& rect, bool rotated, const Size& untrimmedSize)

{

    //这个参数是判断texture是否rotated的，但是具体不是很懂？？

    //将此参数改为true后,图像逆时针反转了90度

    _rectRotated = rotated;



    setContentSize(untrimmedSize);

    setVertexRect(rect);

    //见下

    setTextureCoords(rect);



    //？？

    //需要特别了解_quad

    //变量_unflippedOffsetPositionFromCenter如字面意思没有翻转前离中心点(?)的Vec2

    //目前看到该变量的入口是setSpriteFrame(SpriteFrame*)

    float relativeOffsetX = _unflippedOffsetPositionFromCenter.x;

    float relativeOffsetY = _unflippedOffsetPositionFromCenter.y;



    // issue #732

    if (_flippedX)

    {

        relativeOffsetX = -relativeOffsetX;

    }

    if (_flippedY)

    {

        relativeOffsetY = -relativeOffsetY;

    }



    _offsetPosition.x = relativeOffsetX + (_contentSize.width - _rect.size.width) / 2;

    _offsetPosition.y = relativeOffsetY + (_contentSize.height - _rect.size.height) / 2;



    // rendering using batch node

    if (_batchNode)

    {

        // update dirty_, don't update recursiveDirty_

        setDirty(true);

    }

    else

    {

        // self rendering

        

        // Atlas: Vertex

        float x1 = 0.0f + _offsetPosition.x;

        float y1 = 0.0f + _offsetPosition.y;

        float x2 = x1 + _rect.size.width;

        float y2 = y1 + _rect.size.height;



        // Don't update Z.

        _quad.bl.vertices.set(x1, y1, 0.0f);

        _quad.br.vertices.set(x2, y1, 0.0f);

        _quad.tl.vertices.set(x1, y2, 0.0f);

        _quad.tr.vertices.set(x2, y2, 0.0f);

    }

    

    _polyInfo.setQuad(&_quad);

}



void Sprite::setTextureCoords(Rect rect)

{

    //经过scale_factor缩放过后的rect

    rect = CC_RECT_POINTS_TO_PIXELS(rect);



    Texture2D *tex = _batchNode ? _textureAtlas->getTexture() : _texture;

    if (! tex)

    {

        return;

    }



    float atlasWidth = (float)tex->getPixelsWide();

    float atlasHeight = (float)tex->getPixelsHigh();



    float left, right, top, bottom;



    //if的预编译在tilemap地图中出现黑色缝隙的时候进行使用，具体原理还未懂？？

    //四个float变量是求在(1, 1)的材质中该材质的位置

    if (_rectRotated)

    {

#if CC_FIX_ARTIFACTS_BY_STRECHING_TEXEL

        left    = (2*rect.origin.x+1)/(2*atlasWidth);

        right   = left+(rect.size.height*2-2)/(2*atlasWidth);

        top     = (2*rect.origin.y+1)/(2*atlasHeight);

        bottom  = top+(rect.size.width*2-2)/(2*atlasHeight);

#else

        left    = rect.origin.x/atlasWidth;

        right   = (rect.origin.x+rect.size.height) / atlasWidth;

        top     = rect.origin.y/atlasHeight;

        bottom  = (rect.origin.y+rect.size.width) / atlasHeight;

#endif // CC_FIX_ARTIFACTS_BY_STRECHING_TEXEL



        if (_flippedX)

        {

            std::swap(top, bottom);

        }



        if (_flippedY)

        {

            std::swap(left, right);

        }



        //材质四个点

        _quad.bl.texCoords.u = left;

        _quad.bl.texCoords.v = top;

        _quad.br.texCoords.u = left;

        _quad.br.texCoords.v = bottom;

        _quad.tl.texCoords.u = right;

        _quad.tl.texCoords.v = top;

        _quad.tr.texCoords.u = right;

        _quad.tr.texCoords.v = bottom;

    }

    else

    {

#if CC_FIX_ARTIFACTS_BY_STRECHING_TEXEL

        left    = (2*rect.origin.x+1)/(2*atlasWidth);

        right    = left + (rect.size.width*2-2)/(2*atlasWidth);

        top        = (2*rect.origin.y+1)/(2*atlasHeight);

        bottom    = top + (rect.size.height*2-2)/(2*atlasHeight);

#else

        left    = rect.origin.x/atlasWidth;

        right    = (rect.origin.x + rect.size.width) / atlasWidth;

        top        = rect.origin.y/atlasHeight;

        bottom    = (rect.origin.y + rect.size.height) / atlasHeight;

#endif // ! CC_FIX_ARTIFACTS_BY_STRECHING_TEXEL



        if(_flippedX)

        {

            std::swap(left, right);

        }



        if(_flippedY)

        {

            std::swap(top, bottom);

        }

        //u、v是材质坐标对应的横纵

        _quad.bl.texCoords.u = left;

        _quad.bl.texCoords.v = bottom;

        _quad.br.texCoords.u = right;

        _quad.br.texCoords.v = bottom;

        _quad.tl.texCoords.u = left;

        _quad.tl.texCoords.v = top;

        _quad.tr.texCoords.u = right;

        _quad.tr.texCoords.v = top;

    }

}



//对应上面的setTextureRect()

void Sprite::setSpriteFrame(SpriteFrame *spriteFrame)

{

    // retain the sprite frame

    // do not removed by SpriteFrameCache::removeUnusedSpriteFrames

    if (_spriteFrame != spriteFrame)

    {

        CC_SAFE_RELEASE(_spriteFrame);

        _spriteFrame = spriteFrame;

        spriteFrame->retain();

    }



    //这个getOffset()目前所知道的入口在于SpriteFrame::create(filename, rect, rotated, offset, originSize)

    //offset即是对应此offset，不过scale_factor参数影响。具体在SpriteFrame.cpp分析

    _unflippedOffsetPositionFromCenter = spriteFrame->getOffset();



    Texture2D *texture = spriteFrame->getTexture();

    // update texture before updating texture rect

    if (texture != _texture)

    {

        setTexture(texture);

    }



    // update rect

    _rectRotated = spriteFrame->isRotated();

    setTextureRect(spriteFrame->getRect(), _rectRotated, spriteFrame->getOriginalSize());

}







initWithPolygon特殊一些

bool Sprite::initWithPolygon(const cocos2d::PolygonInfo &info)

{

    Texture2D *texture = Director::getInstance()->getTextureCache()->addImage(info.filename);

    bool res = false;

    if(initWithTexture(texture))

    {

        //多了一个这个参数，？？

        _polyInfo = info;

        setContentSize(_polyInfo.rect.size/Director::getInstance()->getContentScaleFactor());

        res = true;

    }

    return res;

}



void Sprite::updateTransform(void);



void Sprite::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)

{

    //自动裁剪

#if CC_USE_CULLING

    // Don't do calculate the culling if the transform was not updated

    auto visitingCamera = Camera::getVisitingCamera();

    auto defaultCamera = Camera::getDefaultCamera();

    if (visitingCamera == defaultCamera) {

        _insideBounds = ((flags & FLAGS_TRANSFORM_DIRTY)|| visitingCamera->isViewProjectionUpdated()) ? renderer->checkVisibility(transform, _contentSize) : _insideBounds;

    }

    else

    {

        _insideBounds = renderer->checkVisibility(transform, _contentSize);

    }



    if(_insideBounds)

#endif

    {

        //初始化QuadCommand对象，这就是渲染命令，会丢到渲染队列里

        _trianglesCommand.init(_globalZOrder, _texture->getName(), getGLProgramState(), _blendFunc, _polyInfo.triangles, transform, flags);

        renderer->addCommand(&_trianglesCommand);

    }

}



有关batchNode先忽略



//进行debugdraw，数据基于_polyInfo。如果是autoPolygon创建的会更精确，但是作用在那里？是否可以进行碰撞检测?

void Sprite::debugDraw(bool on);

