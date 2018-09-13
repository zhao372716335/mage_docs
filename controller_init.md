Magento Controller 初始化

Mage.php 执行run函数，在680行可以看到

```
self::$_app->run(array(
                'scope_code' => $code,
                'scope_type' => $type,
                'options'    => $options,
            ));
```
打开
app/code/core/Mage/Core/Model/App.php
在run函数可以看到：

```
$this->getFrontController()->dispatch();
```


```
public function getFrontController()
    {
        if (!$this->_frontController) {
            $this->_initFrontController();
        }

        return $this->_frontController;
    }
```
可以看到 下面的init初始化函数执行执行一次，


```
protected function _initFrontController()
    {
        $this->_frontController = new Mage_Core_Controller_Varien_Front();
        Mage::register('controller', $this->_frontController);
        Varien_Profiler::start('mage::app::init_front_controller');
        $this->_frontController->init();
        Varien_Profiler::stop('mage::app::init_front_controller');
        return $this;
    }
```

打开：Mage_Core_Controller_Varien_Front查看
init函数


```
const XML_STORE_ROUTERS_PATH = 'web/routers';

public function init()
    {
        Mage::dispatchEvent('controller_front_init_before', array('front'=>$this));
        /××
         ×
           array (size=3)
            'admin' => 
              array (size=2)
                'area' => string 'admin' (length=5)
                'class' => string 'Mage_Core_Controller_Varien_Router_Admin' (length=40)
            'standard' => 
              array (size=2)
                'area' => string 'frontend' (length=8)
                'class' => string 'Mage_Core_Controller_Varien_Router_Standard' (length=43)
            'install' => 
              array (size=2)
                'area' => string 'frontend' (length=8)
                'class' => string 'Mage_Install_Controller_Router_Install' (length=38)
           ×    下面的$routersInfo 的值打印出来是上面。  
           ×/
        $routersInfo = Mage::app()->getStore()->getConfig(self::XML_STORE_ROUTERS_PATH);

        Varien_Profiler::start('mage::app::init_front_controller::collect_routers');
        foreach ($routersInfo as $routerCode => $routerInfo) {
            if (isset($routerInfo['disabled']) && $routerInfo['disabled']) {
                continue;
            }
            if (isset($routerInfo['class'])) {
                $router = new $routerInfo['class'];
                if (isset($routerInfo['area'])) {
                    $router->collectRoutes($routerInfo['area'], $routerCode);
                }
                $this->addRouter($routerCode, $router);
            }
        }
        Varien_Profiler::stop('mage::app::init_front_controller::collect_routers');

        Mage::dispatchEvent('controller_front_init_routers', array('front'=>$this));

        // Add default router at the last
        $default = new Mage_Core_Controller_Varien_Router_Default();
        $this->addRouter('default', $default);

        return $this;
    }
```














