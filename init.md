

### 初始化

关于Magento的初始化

1.入口文件

1.1web入口：index.php

> 因为是在根目录，因此，很多东西都暴漏在外面，因此需要nginx做限制，
> 让很多文件和文件夹不能直接访问（包括隐藏文件和文件夹）

通过函数：Mage::run($mageRunCode, $mageRunType); 进入到Mage.php

1.2 shell入口： ./shell

里面可以新建通过shell执行的php文件，里面新建的类需要继承类：Mage_Shell_Abstract

里面也是包含Mage.php文件，因此Mage.php是Magento的主类文件


2.Mage.php

Mage类有很多的 `static`变量 和 `static`方法，将magento的各个底层功能挂在到Mage的类静态变量，
作为一种单例模式，将各个功能对象挂到Mage的静态变量中，在初始化的过程中进行组装

查看`Mage.php`的run方法，可以看到下一步的工作是
`Mage_Core_Model_App`这个单例模式对象进行的工作


2.1 打开Mage_Core_Model_App的run方法

初始化模块：`$this->_initModules();`

进行各种初始化

执行controller：`$this->getFrontController()->dispatch();`

然后找到相应的模块的controller进行下一步的执行

3.Layout

经过了magento的初始化，进入controller后，
执行layout初始化，加载最终的xml，通过block的数据和phtml的视图，将数据计算出来


### 功能点

1.xml配置

1.1模块配置文件xml加载
magento的灵活性得益于xml配置，每一个模块中都有相应的模块配置和layout xml配置


magento的xml，是由model: `Mage_Core_Model_Config`
的init函数加载：


```
 public function init($options=array())
    {
        $this->setCacheChecksum(null);
        $this->_cacheLoadedSections = array();
        $this->setOptions($options);
        $this->loadBase();

        $cacheLoad = $this->loadModulesCache();
        if ($cacheLoad) {
            return $this;
        }
        $this->loadModules();
        $this->loadDb();
        $this->saveCache();
        return $this;
    }
```

xml初始化的最终结果写到了变量：`$this->_xml` （类型：Varien_Simplexml_Element）

xml配置文件中有固定语法的重写机制，可以通过这个，在新建模块中重写替换已有的magento功能类

因此在模块的配置文件中的xml文件，最终都会在这个步骤加载完成


1.2 在程序中动态修改xml文件内容

可以通过下面的代码：

```
$store = Mage::app()->getStore();
$store->setConfig('payment/'.$methodNewCode.'/'.$key, $value);
```

通过程序动态的添加xml配置

```
public function setConfig($path, $value)
    {
        if (isset($this->_configCache[$path])) {
            $this->_configCache[$path] = $value;
        }
        $fullPath = 'stores/' . $this->getCode() . '/' . $path;
        Mage::getConfig()->setNode($fullPath, $value);

        return $this;
    }
```

第一步骤加载了配置文件中的配置内容写入到了
Mage_Core_Model_Config对象的`$this->_xml`变量中，
二步骤，我们可以使用程序动态的修改`$this->_xml`变量,
因此，配置的最终更改是由文件，数据库存储，以及程序动态设置而来

```
Mage::getConfig()->setNode($fullPath, $value);
```

2.事件

magento的灵活性得益于事件，在magento的初始化，controller的初始化，以及layout，block
的各个数据的计算过程中都嵌入了大量的event，通过事件机制，就可以在很多的层面修改代码，
满足自己的需求


因为magento event事件传递的是对象，
php在函数中传递对象，传递的是指针，因此，相当于在相应的地方动态插入代码

因此我们可以通过在我们的自定义模块中,添加event，然后在event对应的方法中使用
` Mage::getConfig()->setNode($fullPath, $value);` 来更改xml配置


通过`xml`+`event`，就非常的灵活了，可以在相应的代码执行阶段
更改全局配置，进而改变功能


3.layout

当代码走到控制器，就进入了layout渲染页面的部分

`Mage_Core_Model_Layout`

```
 public function __construct($data=array())
    {
        $this->_elementClass = Mage::getConfig()->getModelClassName('core/layout_element');
        $this->setXml(simplexml_load_string('<layout/>', $this->_elementClass));
        $this->_update = Mage::getModel('core/layout_update');
        parent::__construct($data);
    }
```

初始化是从template中的layout的xml合并起来的，将值作为初始值，然后初始化到layout的类变量中

layout可以通过方法动态的添加创建新的block，然后嵌入到相应的layout 配置中

例子1：直接得到block对应的html
```
$html = $this->getLayout()->createBlock('cms/block')
                ->setBlockId($this->getCurrentCategory()->getLandingPage())
                ->toHtml();

```

例子2：将create的block嵌入都某个block中(下面的代码是嵌入到`content`这个block中)

```
public function indexAction()
{
    //Get current layout state
    $this->loadLayout();
    $block = $this->getLayout()->createBlock(
        'Mage_Core_Block_Template',
        'my_block_name_here',
        array('template' => 'activecodeline/developer.phtml')
    );

    // 从layout中得到一个已有的block，然后将上面创建的加入的写法。
    $this->getLayout()->getBlock('content')->append($block);
    //Release layout stream... lol... sounds fancy
    $this->renderLayout();
}
```

例子3,在layout xml中新建某个handle

```
<checkout_onepage_paymentmethod>
        <remove name="right"/>
        <remove name="left"/>
 
        <block type="checkout/onepage_payment_methods" name="root" output="toHtml" template="checkout/onepage/payment/methods.phtml">
            <action method="setMethodFormTemplate"><method>purchaseorder</method><template>payment/form/purchaseorder.phtml</template>               </action>
        </block>
</checkout_onepage_paymentmethod>
```
然后动态引入到当前

```
protected function _getPaymentMethodsHtml()
    {
        $layout = $this->getLayout();
        $update = $layout->getUpdate();
        $update->load('checkout_onepage_paymentmethod');   // checkout_onepage_paymentmethod 就是上面新建的handle 
        $layout->generateXml();
        $layout->generateBlocks();
        $output = $layout->getOutput();
        return $output;
    }
```

这个和在layout xml中，在当前handle的配置中使用 `<update handle="checkout_onepage_paymentmethod" />`  是一样的。
    

无论那种方式，最终还是更改了初始化配置（确切的说是通过初始化实例化的相应的底层功能对象，譬如Mage_Core_Model_Config）


因此，通过配置文件xml的方式，一切都是配置的方式去对应，
而对于配置本身，也是可以通过程序更改的，因此就可以很方便的更改
配置文件中原有的配置，在加上event机制，我们可以在很多地方通过event
的方式嵌入执行代码，达到我们的目的。

通过xml和event的相互使用，以及xml本身自己的重写机制，可以静态和动态的改变配置，然后画出来整个页面，
这种方式让做扩展插件非常方便，但对开发者就会很费劲，某个功能找到相应的代码就会非常的麻烦，
需要在很多模块的配置中翻来翻去，因为程序是可以动态修改配置的，又造成了找代码的难度。

