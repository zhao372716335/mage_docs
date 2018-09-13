# Magento Model 部分


### Model生成

在Mage.php 的run函数中可以看到某些model，是直接通过class名字new出来的，譬如config

```
protected static function _setConfigModel($options = array())
    {
        if (isset($options['config_model']) && class_exists($options['config_model'])) {
            $alternativeConfigModelName = $options['config_model'];
            unset($options['config_model']);
            $alternativeConfigModel = new $alternativeConfigModelName($options);
        } else {
            $alternativeConfigModel = null;
        }

        if (!is_null($alternativeConfigModel) && ($alternativeConfigModel instanceof Mage_Core_Model_Config)) {
            self::$_config = $alternativeConfigModel;
        } else {
            self::$_config = new Mage_Core_Model_Config($options);
        }
    }
```

一些很底层的对象new出来后，后面就可以通过方法，通过xml的配置，动态找到相应的配置了

`Mage::getModel('xx/yy_zz');`

```
  public static function getModel($modelClass = '', $arguments = array())
    {
        return self::getConfig()->getModelInstance($modelClass, $arguments);
    }
```

也就是：`Mage_Core_Model_Config`的getModelInstance()方法


```
public function getModelInstance($modelClass='', $constructArguments=array())
    {
        $className = $this->getModelClassName($modelClass);
        if (class_exists($className)) {
            Varien_Profiler::start('CORE::create_object_of::'.$className);
            $obj = new $className($constructArguments);
            Varien_Profiler::stop('CORE::create_object_of::'.$className);
            return $obj;
        } else {
            return false;
        }
    }
```

也就是通过函数 $this->getModelClassName($modelClass);得到class的name，然后new出来对象

下面找一下得到class name的函数

```
public function getModelClassName($modelClass)
    {
        $modelClass = trim($modelClass);
        if (strpos($modelClass, '/')===false) {
            return $modelClass;
        }
        return $this->getGroupedClassName('model', $modelClass);
    }
```

通过上面可以看出，如果传递的参数没有`/`，则直接返回，譬如对于
Mage::getModel('Mage_Core_Model_Layout'),直接返回传递的字符串

而对于Mage::getModel('core/layout')，则会调用`$this->getGroupedClassName('model', $modelClass);`


```
/**
     * Retrieve class name by class group
     *
     * @param   string $groupType currently supported model, block, helper
     * @param   string $classId slash separated class identifier, ex. group/class
     * @param   string $groupRootNode optional config path for group config
     * @return  string
     */
    public function getGroupedClassName($groupType, $classId, $groupRootNode=null)
    {
        // 找到group类型节点，譬如 block  model helper等
        // 打开 app/code/core/Mage/Catalog/etc/config.xml 可以看到，早global节点下有blocks  models helpers等节点
        if (empty($groupRootNode)) {
            $groupRootNode = 'global/'.$groupType.'s';
        }
        // 譬如：Mage::getModel('core/layout_element')
        $classArr = explode('/', trim($classId));
        $group = $classArr[0];
        $class = !empty($classArr[1]) ? $classArr[1] : null;
        // 如果之前计算过这个配置节点的class，就会保存到 对象变量$this->_classNameCache中，下次直接返回（注意，这个不是真正的cache，直接一个类变量保存起来值。）
        if (isset($this->_classNameCache[$groupRootNode][$group][$class])) {
            return $this->_classNameCache[$groupRootNode][$group][$class];
        }
        // xml文件合并后，保存到对象$this->_xml中，可以使用下面的方式按照层访问相应的值
        // 如果调用：Mage::getModel('adyen/order_payment')
        // 下面的代码就找到 <config><global><models><adyen>
        $config = $this->_xml->global->{$groupType.'s'}->{$group};

        // First - check maybe the entity class was rewritten
        $className = null;
        if (isset($config->rewrite->$class)) {
            $className = (string)$config->rewrite->$class;
        } else {
            /**
             * Backwards compatibility for pre-MMDB extensions.
             * In MMDB release resource nodes <..._mysql4> were renamed to <..._resource>. So <deprecatedNode> is left
             * to keep name of previously used nodes, that still may be used by non-updated extensions.
             */
            if (isset($config->deprecatedNode)) {
                $deprecatedNode = $config->deprecatedNode;
                $configOld = $this->_xml->global->{$groupType.'s'}->$deprecatedNode;
                if (isset($configOld->rewrite->$class)) {
                    $className = (string) $configOld->rewrite->$class;
                }
            }
        }

        // Second - if entity is not rewritten then use class prefix to form class name
        if (empty($className)) {
            if (!empty($config)) {
                $className = $config->getClassName();
            }
            if (empty($className)) {
                $className = 'mage_'.$group.'_'.$groupType;
            }
            if (!empty($class)) {
                $className .= '_'.$class;
            }
            $className = uc_words($className);
        }

        $this->_classNameCache[$groupRootNode][$group][$class] = $className;
        return $className;
    }
```

在上面代码中已经加入了注释











































