---
layout:     post
title:      "React Native入门"
subtitle:   ""
date:       2016-06-30
author:     "goIntoAction"
header-img: "img/post-bg.jpg"
tags:
    - React Native
---
## React Native入门

### 什么是React Native
React Native是Facebook开发的一种使用JavaScript来开发移动应用的技术，相比Hybrid，它不是使用webview来做界面的载体，而是将React组件映射成当前平台的原生控件，所以性能上比hybrid高很多，不会像Html5界面一样容易卡顿。并且与其他前端技术一样，React Native在开发效率上比原生Android、iOS都高，而且能在不同平台上使用相同的开发方式，天生具有热修复、热更新、插件化的优势，所以React Native一经推出，就迅速成为技术热点。如何搭建环境，运行官方Demo，[官方文档](http://facebook.github.io/react-native/docs/getting-started.html)已经有说明。

### React Native一些重要概念
* 组件

  在React中，使用组件来封装界面模块，整个界面就是一个大组件，里面又套着小组件，形成一颗树。每个组件都有自己的生命周期，组件初始化、被加载、渲染等等都有相应的生命周期函数回调。我们还可以自定义组件，类似于android、iOS中的View。

* State

  在React组件中都有一个state对象用来保存一些状态属性，这些状态属性可以自己根据需求自定义，当这些状态属性被修改的时候，会导致React重新渲染。

* 虚拟DOM

  React在中内存维护了一个虚拟的DOM，当state改变时，React都会重新构建整个虚拟DOM树，然后React将当前整个虚拟DOM树和上一次的虚拟DOM树使用diff算法做对比，然后找出需要更新的地方，再去更新界面，这也是React比较高效的原因之一。

* props

  在React组件中传递数据、回调函数，是通过props来进行的，比如父组件将一个名为title字符串传给子组件，子组件只需要写this.props.title就能取得。至于如何传给子组件下面再讲

### React的生命周期
跟Android的Activity一样，React的组件也有自己的生命周期回调，生命周期回调已经帮我们规划好哪里应该做什么，所以我们要懂它，才能更合理的开发。

![](/img/react_native/rn_life_cycle.png)

React Native的生命周期可以分为四个部分
* 创建阶段，一个组件只会创建一次。
* 初始化和加载，在这里完成初始化和第一次渲染，最后被加载。
* 组件运行，此时props、state可能改变，导致重新渲染。
* 组件卸载，在这里做一些资源回收工作。

下面就来看具体每一个方法。
##### getDefaultProps

这个方法在只会在createClass被调用一次，组件被再次使用不会再调用，所以不能使用具体对象实例的东西。前面我们讲过父组件会通过props传递一些属性，在此方法里，组件也为自身设置props，这方法的返回值将为被缓存到props中。这个方法较少使用。

##### getInitialState
这个方法主要是设置初始化状态，在这里的返回值就是上面说的state。比如设置一个loading的布尔值，如何在渲染时根据loading来决定是否显示加载动画。
```
var MessageBox = React.createClass({
  getInitialState: function() {
    return {loading:true};
  },
......
}
```

#### componentWillMount
在组件被加载前调用，生命周期里只会调用一次，在这里可以设置一些状态，做一些准备工作。

#### render
render是React组件的渲染方法，初始化时在componentWillMount后调用，此方法的返回值就是组件显示的东西。


#### componentDidMount
组件加载完成后调用，生命周期里只会调用一次，此时可以去做一些异步获取网络数据等操作。


#### componentWillReceiveProps
当组件收到新的props时就会调用，此回调有一个参数——nextProps，就是新的props。在生命周期回调函数中，这是唯一一个可以setState的地方，在此处setState不会导致组件重新渲染。
```
componentWillReceiveProps: function(nextProps) {
  this.setState({

  });
}
```
#### shouldComponentUpdate
在state被改变的时候，可以在这里做一些判断，来决定要不要重新渲染组件

#### componentWillUpdate
当需要重新渲染组件时，此方法就会被回调。这个函数执行完后，又会调用到render。

#### componentDidUpdate
当重新渲染完成后调用。

#### componentWillUnmount
当组件被卸载时调用，可以做一些资源回收工作。

### JSX
JSX是一个的JavaScript语法扩展，它能使你像写XML一样去写JavaScript，最终写出来的代码会被编译成纯Javascript代码。这里为什么要说JSX呢，因为React官方推荐使用它来开发`UI界面`，一般我们会在组件的render函数使用到它。如果还使用createElement来写，不仅代码繁琐，创建嵌套也不清晰，不利于维护，这些问题使用JSX后能改善很多。说了它的好处后，我们来见下它的庐山真面目。
```
render(){
    return <View style={styles.headerWrapper}>
    <Image source={url} style={{flex: 1}}/>
    <View style={styles.editorWrapper}>
      <Text style={styles.imageEditors}>{title}</Text>
    </View>
</View>;
}
```

这就是一小段JSX代码，在JSX中如果要使用到JavaScript表达式，需要使用{}括起来。像style、source这些就是此标签的属性，跟XML一样。有朋友会问为什么里面有个`{{flex: 1}}`，这里我们要把的`{flex: 1}`看成一个JavaScript对象，外面再加一层{}给属性赋值。
### CSS和FlexBox
React Native中可以使用CSS来设置样式，使用方法如下。
```
<View style={styles.test}/>
 const styles = StyleSheet.create({
    test:{
        height:40,
        width:80,
        borderWidth: 1,
        borderColor: 'red',
        backgroundColor :'gray'
      },
 });
```
![css](/img/react_native/rn_css.png)

CSS没什么特别的，之前应该都接触过。接下来说FlexBox，这是一个布局引擎，是ReactNative中使用的布局方式，详细的语法可以看另一篇文章FlexBox基础语法。使用FlexBox也很简单，因为它是CSS3新增的内容，使用我们还是像使用CSS一样去使用它。
```
<View style={styles.test}/>
 const styles = StyleSheet.create({
    test:{
         flex:1,
         flex-direction:column,
        height:40,
        width:80,
      },
 });
```
### 映射和封装
React Native官方说组件会被映射成原生控件，我们先跑起官方Demo来看下会被映射成什么。App启动后我们打开hieararchyviewer查看应用，发现确实是映射成自定义的原生控件。

![](/img/react_native/rn_1.png)

那么它是如何进行映射？这个问题我们可以先查看官方文档中封装原生控件的教程。要把一个原生控件封装成ReactNative组件，有下列几步
1.在原生代码新建一个类继承SimpleViewManager，SimpleViewManager是一个带泛型的抽象类，继承它的时候需要传入一个View的子类型，代表要被封装的原生类。
```
public class TextViewManager extends SimpleViewManager<ReactImageView>
```
2.实现方法,有两个方法必须实现，第一个是getName,这个方法的返回值就是在JavaScript那边使用此类的名称。createViewInstance需要返回一个原生类的对象，会被动态添加到界面上。

	 @Override
    public String getName() {
        return "TextView";
    }

    @Override
    protected TextView createViewInstance(ThemedReactContext reactContext) {
        return new TextView(reactContext);
    }
    
3.暴露接口，JavaScript中要给原生控件设置属性，需要我们在原生中将接口暴露给它。
```
@ReactProp(name="text")
public void setText(TextView textView, @Nullable String text) {
    textView.setText(text);
}
```
 4.注册组件，JavaScript要使用这个组件，还需要将其注册到ReactInstanceManager，先新建一个ReactPackage的实现类。
```
public class AppReactPackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Arrays.<ViewManager>asList(
                new TextViewManager());
    }
}
```
里面有三个方法，createNativeModules是注册NativeModule，像okhttp之类的库，我们可以在此注册，createJSModules是注册JSModules，createViewManagers才是注册UI控件的。我们在这个方法里把TextViewManager封装到List里并返回。
接着在MainActivity里，找到getPackages方法，把我们的AppReactPackage添加进去。
```
@Override
protected List<ReactPackage> getPackages() {
    return Arrays.<ReactPackage>asList(
        new MainReactPackage(),new AppReactPackage()
    );
}
```
 getPackages会ReactActivity.createReactInstanceManager中被调用，然后添加到ReactInstanceManager中。
 接着我们创建一个TextView.js文件，封装一个JavaScript层可以使用的组件。
```
import { PropTypes } from 'react';
import { requireNativeComponent } from 'react-native';

var iface = {
  name: 'TextView.js',
  propTypes: {
    text: PropTypes.string,
  },
};

module.exports = requireNativeComponent('RCTTextView', iface);
```
如果控件需要触控操作，还需要为其添加映射，这个请查看官方文档。到这里应该就不难明白映射的过程，JavaScript层的UI组件都是原生组件的封装，React框架去解析JavaScript遇到这种组件，就会调用原生去创建相应的View，然后动态添加到父View中。

#### Java与JavaScript间的通讯

上面说到Java的组件要注册才能给JavaScript使用，为什么要注册？这个问题涉及到React Native中Java与JavaScript间的通讯机制，我们知道他们间是无法直接调用的，React在他们之间搭了座桥，调用的方式将{moduleID,methodID,args}传到Bridge中，由Bridge去调用对应的方法，这就要求双方的Module都要在Brigde中注册。

![](/img/react_native/framework.jpg)

ReactInstanceManager是一个重要的类，负责构建React环境，我们把ReactPackage添加到它里面，它又创建了CatalystInstance，CatalystInstance里面维护两个对象NativeModuleRegistry和JSModuleRegistry就是两张Module的注册表。

### 一些常用控件介绍
#### Navigator

Navigator这个组件是React中用来做场景间切换，Android中的Activity是交给AMS管理的，而React Native中是交给Navigator，而它只是给普通组件，不像AMS是一个系统服务，这跟iOS中的UINavigationController有点像。
```
render () {
    // return (<HistoryList/>);
    return (
      <View style = {styles.container}>
        <StatusBar
          backgroundColor='transparent'
          translucent/>
        <Navigator
          ref={component => this.navigator = component}
          initialRoute={{
            component: HomePage
          }}
          configureScene={(route, routeStack) => Navigator.SceneConfigs.FloatFromBottom}
          renderScene={(route, navigator) => {
          return <route.component navigator={navigator} {...route} {...route.passProps}/>
        }}/>
      </View>
    )
  }
```
函数：

* getCurrentRoutes()    该进行返回存在的路由列表信息
* jumpBack()    该进行回退操作  但是该不会卸载(删除)当前的页面
* jumpForward()    进行跳转到相当于当前页面的下一个页面
* jumpTo(route)    根据传入的一个路由信息，跳转到一个指定的页面(该页面不会卸载删除)
* push(route)     导航切换到一个新的页面中，新的页面进行压入栈。通过jumpForward()方法可以回退过去
* pop()   当前页面弹出来，跳转到栈中下一个页面，并且卸载删除掉当前的页面
* replace(route)   只用传入的路由的指定页面进行替换掉当前的页面
* replaceAtIndex(route,index)     传入路由以及位置索引，使用该路由指定的页面跳转到指定位置的页面
* replacePrevious(route)    传入路由，通过指定路由的页面替换掉前一个页面
* resetTo(route)  进行导航到新的界面，并且重置整个路由栈
* immediatelyResetRouteStack(routeStack)   该通过一个路由页面数组来进行重置路由栈
* popToRoute(route)   进行弹出相关页面，跳转到指定路由的页面，弹出来的页面会被卸载删除
* popToTop()  进行弹出页面，导航到栈中的第一个页面，弹出来的所有页面会被卸载删除

属性：

* initialRoute 初始化路由，这个路由对象可以自己随便定义，最后会作为参数route传入renderScene中。
* renderScene 返回值就是要显示的界面，route是initialRoute或者push传进来的，这里自己定义一套规则，能判断出要显示那个界面就行，可以像例子中一样直接把要显示的组件传进来，也可以传进来一个flag，然后在这里判断显示那个组件。
* configureScene 可以定义切换时候的动画。React Native中已经有自带的效果，FloatFromBottom就是从下往上切入。
* navigationBar 可以配置iOS顶部导航栏。

#### Text、TextInput
Text相当于Android中TextView，TextInput相当于EditText。

Text的常用属性

* allowFontScaling 这个是iOS用来设置字体是否自动缩放的
* numberOfLines 设置行数
* onLayout 组件挂载或者布局变化回调
* onPress 当点击时回调

此外Text还有一些特有的Style

* color:字体颜色
* fontFamily 字体名称
* fontSize  字体大小
* fontStyle   字体风格(normal,italic)
* fontWeight  字体粗细权重("normal", 'bold', '100', '200'......)
* textShadowOffset 设置阴影效果{width: number, height: number}
* textShadowRadius 阴影效果圆角       9..textShadowColor 阴影效果的颜色
* letterSpacing 字符间距            11.lineHeight 行高
* textAlign   文本对其方式("auto", 'left', 'right', 'center', 'justify')
* textDecorationLine  横线位置 ("none", 'underline', 'line-through', 'underline line-through')
* textDecorationStyle   线的风格("solid", 'double', 'dotted', 'dashed')
* textDecorationColor  线的颜色       16.writingDirection  文本方向("auto", 'ltr', 'rtl')

TextInput的常用属性

* autoCapitalize 是否自动修改为大写（'none', 'sentences', 'words', 'characters'）
* autoCorrect 是否拼写检查
* autoFocus 是否自动获取焦点
* blurOnSubmit 是否自动提交，为true的话，回车会触发onSubmitEditing，而不换行
* defaultValue 默认值
* editable 是否可编辑
* keyboardType 键盘类型（"default", 'numeric', 'email-address', "ascii-capable", 'numbers-and-punctuation', 'url', 'number-pad', 'phone-pad', 'name-phone-pad', 'decimal-pad', 'twitter', 'web-search'）
* maxLength 最大字符数
* multiline 是否多行
* onBlur 失去焦点回调
* onChange 内容变化回调
* onChangeText 内容变化回调，内容作为参数传递
* onEndEditing 编辑结束回调
* onSubmitEditing 提交回调，multiline={true}时不可用
* placeholder 占位提示字符，相当于hint
* selectionColor 输入框高亮时的颜色
* value 文本内容
* secureTextEntry 密码输入

TextInput的方法

* isFocused 是否获取了焦点
* clear 清空内容

#### TouchableHighlight
这个组件的作用是当按下的时候颜色会产生变化，主要用来包裹其他组件
```
<TouchableHighlight onPress={this._onPressButton}>
  <Image
    style={styles.button}
    source={require('./button.png')}
  />
</TouchableHighlight>
```
 常用属性

 * onLongPress 长按回调
 * onPress 点击回调
 * activeOpacity 视图在被触摸时的透明度
 * underlayColor 触摸时的颜色

#### ListView
 相当于Android中的ListView
```
<ListView
  dataSource={this.state.dataSource}
  renderRow={(rowData) => <Text>{rowData}</Text>}
/>
```
常用属性

* dataSource 数据源，ListView.DataSource类型的
* renderRow 每个item的渲染方法
* initialListSize 加载组件是渲染多少项
* pageSize 每帧渲染的行数

列表的点击事件与Android的RecycleView有点像，需要在每个item做，所以一般都会把item包裹在TouchableHighlight里

### 使用ES6+语法

ES全称是ECMAScript，是一种语言标准规范，JavaScript就是它的一种具体实现。ES6+就是ES6以后的版本，因为ES6做了很多改进，但目前很浏览器还没全面支持ES6，幸好React可以支持，所以建议使用ES6+。

举几个ES6的例子.

1. 声明类，以前JavaScript类的写法很怪异，现在终于支持class关键字
```
class Photo extends React.Component {
  handleDoubleTap(e) { … }
  render() { … }
}
```
2. 箭头函数
```
//es5
var add = function(a,b) {
    return a + b;
}
//es6
var add = (a,b) => a+b
```
3. 解构，能快速将一个对象的属性赋值给一个标签。例如下面这两种写法是相等的
```
var attrs = {name: 'xxx'}
<View name={attrs. name}/>
<View {...attrs}/>//解构
```
4. 块级变量，以前var声明的变量是方法级的，在es6中增加了块级变量，用let声明。

这里只是举例说明，其他es6+的好处请亲身体验。

### 总结

React Native还是比较容易入门的一种技术，开发效率也高，又有热更新这种非常吸引人的优点，确实需要我们做原生开发的去了解一下。就目前来说，一个大项目全用React Native还不现实，目前大多数是用它来做一些更新比较频繁的界面，比如促销活动界面。
