Scene

继承：Node

构造函数：
	创建了一个camera
	创建一个自定义事件监听(应是用来做正交变化的)

析构函数：
	移除了这个自定义事件监听

//默认是winsize大小
bool Scene::initWithSize(const Size& size)
{
    setContentSize(size);
    return true;
}

//移除所有子节点
void Scene::removeAllChildren()
{
	//防止camera被移除先retain一下,
	//等child都移除了,再release。
    	if (_defaultCamera)
        _defaultCamera->retain();
    
    	Node::removeAllChildren();
    	
    	if (_defaultCamera)
    	{
		addChild(_defaultCamera);
		_defaultCamera->release();
    	}
}



//获得所有显示的camera,并对子类进行visit和draw,即Node::visit()
void Scene::render(Renderer* renderer);

const std::vector<Camera*>& Scene::getCameras();

//自定义监听事件中会调用的
void Scene::onProjectionChanged(EventCustom* event)
{
    if (_defaultCamera)
    {
        _defaultCamera->initDefault();
    }
}

//等看physics的时候一起看