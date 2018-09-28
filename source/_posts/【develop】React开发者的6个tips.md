---
title: 【develop】React开发者的6个tips
date: 2018-09-28 19:57:02
categories: develop
tags:
    - React
---

## Tips

* 使用函数式组件
* 保持组件尽量小
* 合适处理”this”
* "setState”中使用函数，而不是对象
* 使用”prop-types”
* 浏览器安装”React Developer Tools”

<!--more-->

### 使用函数式组件

* 更少的代码
* 更容易理解
* 无状态
* 容易分解出更小的组件

### 保持组件尽量小

* 更容易阅读、测试、重复使用

    ```javascript
    class Comment extends Component {
    render() {
        return (
            <div className="Comment">
                <div className="UserInfo">
                    <img className="Avatar">
                        src={this.props.user.avatarUrl}
                        alt={this.props.user.name}
                    />
                <div className="User-Name">
                    {this.props.user.name}
                </div>
                <div className="Comment-text">
                    {this.props.text}
                </div>
                <div className="Comment-date">
                    {formatDate(this.props.date)}
                </div>
            </div>
        );
    }
    }
    ```


分解出更小的组件

```javascript
function UserInfo(props){
    return (
        <div className="UserInfo">
            <Avatar user={props.user}>
            <div className="User-Name">
                {props.user.name}
            </div>
    );
}

function Avatar(props){
    return (
        <img className="Avatar">
                src={props.user.avatarUrl}
                alt={props.user.name}
        />
    );
}
```

### 合适处理"this"

使用ES6，组件类不会自动绑定内部函数。

* 渲染时绑定或使用 arrow function

    ```javascript
    render() {
        return (
            <button onClick={this.logMessage.bind(this)}/>
            <button onClick={() => this.logMessage()}/>
        );
    }
    ```

* 构造时绑定

    ```javascript
    constructor(props) {
        super(props);
        this.logMessage = this.logMessage.bind(this);
    }
    
    render() {
        return (
            <button onClick={this.logMessage}/>
        );
    }
    ```

* 函数定义arrow function

    ```javascript
    logMessage = () => {
        console.log("test")
    }
    
    render() {
        return (
            <button onClick={this.logMessage}/>
        );
    }
    ```

### "setState"中使用函数，而不是对象

```javascript
this.setState({correctData: !this.state.correctData})

this.setState((prevState, props) => {
    return {correctData: !prevState.correctData});
}
```

使用函数第一个参数获取prevState，第二个参数porps。

### 使用"prop-types"

"prop-types"是一个库，检查props的类型

```javascript
import PropTypes from "prop-types";

class Welcome extends Component {
    reder() {
        return (
            <h1>Hello, {this.props.name}</h1>
        );
    }
}

Welcome.propTypes = {
    name: propTypes.string.isRequired
}
```

### 浏览器安装"React Developer Tools"


### Ref:

* [<https://www.youtube.com/watch?v=xa-_FIy2NgE>](https://www.youtube.com/watch?v=xa-_FIy2NgE)