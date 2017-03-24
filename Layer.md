Layer：

继承node



构造函数，多了几个特定的参数，并进行初始化

layer的锚点为Vec2(0.5, 0.5)，但是因为开启了_ignoreAnchorPointForPosition，

.所以可以理解为锚点为Vec2::Zero



bool Layer::init()

{

    Director * director = Director::getInstance();

    //contentSize为WinSize大小，也即是说放在layer上的需要做适配

    setContentSize(director->getWinSize());

    return true;

}



Layer貌似没有太多可以东西，很多东西都是不提倡使用的

如setTouchEnabled，setKeyPad..等弃用函数，但是用依然还是可以用的。



__LayerRGBA之后和__NodeRGBA一起学习



LayerColor:

继承Layer、BlendProtocol



可以改变宽高的参数



virtual void draw(Renderer *renderer, const Mat4 &transform, uint32_t flags) override;

virtual void setContentSize(const Size & var) override;

bool initWithColor(const Color4B& color, GLfloat width, GLfloat height);

void onDraw(const Mat4& transform, uint32_t flags);

virtual void updateColor() override;



LayerGradient(大概是颜色梯度) 和 LayerMultiplex  优先级放后面