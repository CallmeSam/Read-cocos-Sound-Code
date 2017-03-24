# Node

## 继承：ref

## 成员函数：
* [void Node::setLocalZOrder(.)](#1)

* [void Node::addChildHelper(...)](#1.1)

* [void Node::onEnter()](#1.2)

* [virtual ActionManager\* getActionManager()](#1.3)

* [virtual void enumerateChildren(..)](#1.4)

* [virtual void setParent(.)](#1.5)

* [Rect Node::getBoundingBox()](#1.6)

* [virtual void removeFromParentAndCleanup(.) 和 void Node::detachChild(...)](#1.7)

* [void Node::reorderChild(..) 和 Node::sortAllChildren()](#1.8)

* [inline T getChildByName(.) const](#1.9)

* [还未明白](2.0)
 


<br>
<h1 id="1"></h1>

><font color="#4590a3" size = "3px"> **void Node::setLocalZOrder(int z)**</font>

	void Node::setLocalZOrder(int z)
	{
	    //如果 z!=_localZOrder 则_parent->reorderChild(this, z)
	    ...
	    _eventDispatcher->setDirtyForNode(this);
	}

   >**_localZOrder**、**_globalZOrder**、**_orderOfArrival**

    cocos渲染顺序是依据下列三个成员变量的值的大小来进行渲染：

*   _globalZOrder:  优先级最高，值越低越先渲染。

*   _localZOrder  :  是用来和同级节点(siblings兄弟姐妹)进行渲染顺序区分，同样是越小越先渲染

*   _orderOfArrial:  是用来在同级节点且_localZOrder相同的情况下，越小的越先渲染。这个值来自Node中的静态变量s_globalOrderOfArrival，每用一次自加1。

若三个值都相同的情况下(基本不太可能吧)，绘制的顺序是不能预测的(在一个博客上看到的)。

<br />
<h1 id="1.1"></h1>

><font color="#4590a3" size = "3px">**void Node::addChildHelper(Node\* child, int localZOrder, int tag, const std::string &name, bool setTag)**</font>

	//addChild会调用
	void Node::addChildHelper(Node* child, int localZOrder, int tag, const std::string &name, bool setTag)
	{
   		if (_children.empty())
    	{
	        //把_children(Vector)容量设为4
	        this->childrenAlloc(); 
	    }
    
	    //pushback到_children中(自动retain)
	    this->insertChild(child, localZOrder);
	    
	    if (setTag)
	        child->setTag(tag);
	    else
	        child->setName(name);
	    
	    child->setParent(this);
	    child->setOrderOfArrival(s_globalOrderOfArrival++);
	    
	    //如果此时的_parent在运行状态则加入的child也会onEnter
	    //如果该_parent过度完成，则在onEnter完成之后也会调用finish
	    if( _running )
	    {
	        child->onEnter();
	        // prevent onEnterTransitionDidFinish to be called twice when a node is added in onEnter
	        if (_isTransitionFinished)
	        {
	            //应是：调用了cocos的transition
	            child->onEnterTransitionDidFinish();
	        }
	    }
	    
	    //如果允许颜色或者透明度传递，则按照父节点的透明度百分比进行调整
	    //此刻 数值 * 父数值 ／ 225
	    if (_cascadeColorEnabled)
	    {
	        updateCascadeColor();
	    }
	    
	    if (_cascadeOpacityEnabled)
	    {
	        updateCascadeOpacity();
	    }
	}

<br/>
<h1 id="1.2"></h1>

><font color="#4590a3" size = "3px">**void Node::onEnter()**</font>

	//删除了script-binding和component(作用未知).  
	//onExit和onEnter与此同理
	void Node::onEnter()
	{
	    //如果用户设置了回调，则在onEnter的时候会进行回调
	    if (_onEnterCallback)
	        _onEnterCallback();
	
	    //这时候还没有进行Entertransition
	    _isTransitionFinished = false;
	    for( const auto &child: _children)
	        child->onEnter();
	
	    //计时器开启、动作开启、时间监听开启
	    //默认的的_scheduler、_actionManager、_eventDispatcher都是来自Director
	    this->resume();
	    _running = true;
	}

>**onEnter**和**onExit**

    onEnter和onExit应是标志着改节点是否在此刻运行的舞台上，即_parent中_running参数是true或false。

*   onEnter:    \_running参数被设置为true，_scheduler, _actionManager, _eventDispatcher都继续工作。
*   onExit :    反之。是暂停工作。


<br/>
<h1 id="1.3"></h1>

><font color="#4590a3" size = "3px"> **virtual ActionManager\* getActionManager()**</font>

    virtual ActionManager* getActionManager() { return _actionManager; }
    virtual const ActionManager* getActionManager() const { return _actionManager; }
>c++多态

    牵扯到的是c++多态问题virtual中的const(尾部修饰的const)可以看作为一个参数。
    即，若子类若想重写(override)，也需要添加const，否则视为覆盖(hide)。
    尾部修饰符的const的意思：类成员变量不可改变。

<br />
<h1 id="1.4"></h1>

><font color="#4590a3" size = "3px"> **virtual void enumerateChildren(const std::string &name, std::function<bool(Node\* node)> callback) const**</font>

	//基本的正则匹配  并不会使用“/..”
	virtual void enumerateChildren(const std::string &name, std::function<bool(Node* node)> callback) const;
	bool doEnumerate(std::string name, std::function<bool (Node *)> callback) const;
	bool doEnumerateRecursive(const Node* node, const std::string &name, std::function<bool (Node *)> callback) const;


<br />
<h1 id="1.5"></h1>

><font color="#4590a3" size = "3px"> **virtual void setParent(Node\* parent)**</font>

	//感觉这个函数应该属于内部函数，函数内部只是_parent指针指向了该node
	//即父类容器并没有添加该子类，前父类也并没有移除掉该node，通过改节点进行的一系列与父类相关的操作都是无用的
	virtual void setParent(Node* parent);

<br/>
<h1 id="1.6"></h1>

><font color="#4590a3" size = "3px"> **Rect Node::getBoundingBox() const**</font>

	Rect Node::getBoundingBox() const
	{
	    Rect rect(0, 0, _contentSize.width, _contentSize.height);
	    //进行仿射变换  即对rect进行 缩放和平移
	    return RectApplyAffineTransform(rect, getNodeToParentAffineTransform());
	}
  
<br />
<h1 id="1.7"></h1>

><font color="#4590a3" size = "3px">**virtual void removeFromParentAndCleanup(bool cleanup)**</font>
	
	//判断其父类是否存在该子类，存在则调用detachChild。
	virtual void removeFromParentAndCleanup(bool cleanup);

	void Node::detachChild(Node *child, ssize_t childIndex, bool doCleanup)
	{
	    // IMPORTANT:
	    //  -1st do onExit
	    //  -2nd cleanup
	    if (_running)
	    {
	        child->onExitTransitionDidStart();
	        child->onExit();
	    }
	
	    // If you don't do cleanup, the child's actions will not get removed and the
	    // its scheduledSelectors_ dict will not get released!
	    if (doCleanup)
	    {
	        //暂停所有子类动作管理器和计时器
	        child->cleanup();
	    }
	    // set parent nil at the end
	    child->setParent(nullptr);
	    //如果存在则release
	    _children.erase(childIndex);
	}

<br />
<h1 id="1.8"></h1>

><font color="#4590a3" size = "3px">**void Node::reorderChild(Node \*child, int zOrder)**</font>

	void Node::reorderChild(Node *child, int zOrder)
	{
	    CCASSERT( child != nullptr, "Child must be non-nil");
	    _reorderChildDirty = true;
	    //避免相同_orderOfArrival
	    child->setOrderOfArrival(s_globalOrderOfArrival++);
	    child->_localZOrder = zOrder;
	}


	void Node::sortAllChildren()
	{
	    //addChild(insertChild)和reorderChild 都会使得_reorderChildDirty
	    if (_reorderChildDirty)
	    {
	        //localZOder越小越在前面
	        std::sort(std::begin(_children), std::end(_children), nodeComparisonLess);
	        _reorderChildDirty = false;
	    }
	}

<br />
<h1 id="1.9"></h1>

	//这个方法挺好用的
	//getChildByTag<Sprite*>(1);将child直接转成需要的类型
	template <typename T>
	inline T getChildByName(const std::string& name) const { return static_cast<T>(getChildByName(name)); }



<br />
<h1 id="2.0"></h1>

<font color="#DAA520" size = "3px">不会的</font>

    有关节点的变换(visible, position, skewX...)的一类函数会与\_transformUpdated、_transformDirty、_inverseDirty这几个变量相关

    _actionManager去看ActionManager类.

    _scheduler去看Scheduler类.



    //有关openGL的东西

    //先visit再draw

	virtual void visit(Renderer *renderer, const Mat4& parentTransform, uint32_t parentFlags);
	
	virtual void visit() final;
	
	
	
	virtual void draw(Renderer *renderer, const Mat4& transform, uint32_t flags);
	
	void Node::draw()
	{
	    //获得渲染器
	    auto renderer = _director->getRenderer();
	    draw(renderer, _modelViewTransform, true);
	}


	
	virtual void setNormalizedPosition(const Vec2 &position);
	
	virtual void setGLProgram(GLProgram *glprogram);
	
	virtual void setGLProgramState(GLProgramState *glProgramState);
	
	
	virtual void updateTransform();
	
	virtual const Mat4& getNodeToParentTransform() const;
	
	virtual AffineTransform getNodeToParentAffineTransform() const;
	
	virtual void setNodeToParentTransform(const Mat4& transform);
	
	virtual const Mat4& getParentToNodeTransform() const;
	
	virtual AffineTransform getParentToNodeAffineTransform() const;
	
	virtual Mat4 getNodeToWorldTransform() const;
	
	virtual AffineTransform getNodeToWorldAffineTransform() const;
	
	virtual Mat4 getWorldToNodeTransform() const;
	
	virtual AffineTransform getWorldToNodeAffineTransform() const;



返回值为Mat4的应是用于node和world坐标系的转化.


    void setAdditionalTransform(Mat4* additionalTransform);
    
    void setAdditionalTransform(const AffineTransform& additionalTransform);



关于component


物理引擎类先略过



    _displayedOpacity  最终显示的不透明度(自身的透明度 * parent的最终不透明度)

    _realOpacity          自身的透明度

	void Node::updateDisplayedOpacity(GLubyte parentOpacity)
	{
	    //根据父类的透明度比例得出
	    _displayedOpacity = _realOpacity * parentOpacity/255.0;
	
	    updateColor();
	
	    if (_cascadeOpacityEnabled)
	    {
	        for(const auto& child : _children)
	
	        {
	            //并传递到child
	            child->updateDisplayedOpacity(_displayedOpacity);
	        }
	    }
	}


	virtual void setOpacityModifyRGB(bool value) {CC_UNUSED_PARAM(value);}
	
	virtual bool isOpacityModifyRGB() const { return false; };
	
	unsigned short getCameraMask() const { return _cameraMask; }
	
	virtual void setCameraMask(unsigned short mask, bool applyChildren = true);
	
	/// Convert cocos2d coordinates to UI windows coordinate.
	
	Vec2 convertToWindowSpace(const Vec2& nodePoint) const;
	
	Mat4 transform(const Mat4 &parentTransform);
	
	uint32_t processParentFlags(const Mat4& parentTransform, uint32_t parentFlags);
	
	//check whether this camera mask is visible by the current visiting camera
	
	bool isVisitableByVisitingCamera() const;
	
	// update quaternion from Rotation3D
	
	void updateRotationQuat();
	
	// update Rotation3D from quaternion
	
	void updateRotation3D();





漏了一个类在node下方
















