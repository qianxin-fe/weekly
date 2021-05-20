
# if...else..优化的几种方式

一般我们在业务中会遇到很多判断，if...else...满天飞，甚至很多嵌套。这时候代码就有很大的优化空间，这儿我就根据自己遇到的和网友们的智慧，总结一下if...else..优化的几种常用方式。

## return提升

* 单条if...else...语句
  我们业务中经常出现这样的代码：

  ```js
  function test(user) {
    if (user) {
      // 执行其他逻辑
    } else {
      return false
    }
  }
  ```

  就是当user为真值时，才执行其他业务逻辑，因为后面是return，我们就可以将return提升，优化一下：

  ```js
  function test(user) {
    if (!user) return false;
    // 执行其他逻辑
  }
  ```

  这样我们就少了一个if分支，代码也更简洁。
* 多层if嵌套
  我们业务中往往会有很复杂的数据结构，每一个字段都需要校验到，不是简单的判断就可以的。我们来看看下面这段代码：

  ```js
    const parmas = {
      user: {
        name: 'Tom',
        job: {
          name: 'web',
          trade: 'it',
          contents: []
        }
      }
    };
    function valdateData(user) {
      if (user) {
        if (user.job) {
          if (user.job.name) {
            if (user.job.contents && user.job.contents.length) {
              // 执行其他代码
              return true;
            } else {
              return '请输入工作内容';
            }
          } else {
            return '请输入职位名称';
          }
        } else {
          return '请填写职位信息';
        }
      } else {
        return '请填写用户信息';
      }
    }
  ```

  上面一堆if...else...,看着就让人难受。我们先用return提升优化一下。

  ```js
  function valdateData(user) {
    if (!user) return '请填写用户信息';
    if (!user.job) return '请填写职位信息';
    if (!user.job.name) return '请输入职位名称';
    if (!user.job.contents || !user.job.contents.length) return '请输入工作内容';
    // 执行其他代码
    return true;
  }
  ```

  这样是不是更加简洁了呢。

## 使用各种操作符

* 单条if...else...语句
  * 使用&&操作符，去掉不必要的if分支

    ```js
      // 单个if语句
      let flag = true;
      if (flag) {
        test();
      }
      function test() {
        console.log('flag为true时执行该函数');
      }

      // && 优化
      flag && test();
    ```

  * 使用||操作符
  
    ```js
    if (!flag) {
      test();
    }
    flag || test()
    ```

  * 使用??空值合并运算符：
  我们来回顾一下ES2020新增的*??*空值合并运算符：

    ```js
      activeValue ?? defaultValue
      // 等价于
      activeValue !== null && activeValue !== undefined ? activeValue : defaultValue
    ```

    ```js
    if (flag===null || flag === undefined) {
      test();
    }
    flag ?? test()
    ```

  * 三目操作符

    ```js
      if (flag) {
        test1()
      } else {
        test2()
      }
      // 使用三目操作符

      flag ? test1() : test2()
    ```

* 对象深入判断
  * 使用可选链操作符?.
  我们来看一个常见的报错：

    ```js
      function test(obj) {
        if (obj.a) {}
      }
      test()
      // TypeError: Cannot read property 'a' of undefined
    ```

    这个报错相信大家都不陌生。
    如果我们直接这样写的话，很可能会报错。然后我们为了严谨性，就会判断obj是否为undefined或者null

    ```js
      function test(obj) {
        if (obj && obj.a) {}
      }
      test()
      // 使用可选链

      function test(obj) {
        if (obj?.a) {}
      }
      test()
      // obj?.a 等价于 (obj !== undefined && obj !== null) ? obj.a: undefined
    ```

    这样我们的代码就不报错了，看起来也简洁了。这种简单的对象看起来简化就还好，没有特别明显，那我们来看看下面这个例子。
    有这样一个数据:

    ```js
      const parmas = {
        user: {
          name: 'Tom',
          job: {
            name: 'web',
            trade: 'it',
            contents: []
          }
        }
      };
    ```

    业务需求，需要对工作名称修改，或做一些操作。一般情况下，为了代码的严谨性我们是这样的：

    ```js
      if (parmas && parmas.user && parmas.user.job && parmas.user.job.name) {
        // 业务逻辑
      }
    ```

    一长串的条件，看起来真难受，写出来也是重复的一大堆。官方也是听到了社区的这些声音，在ES2020就新增了一个可选链操作符?.。嗯，用了之后，真香！上面的代码就可以简化：

    ```js
      if (parmas?.user?.job?.name) {
        // 业务逻辑
      }
    ```

    这样看起来真是赏心悦目，而且也清晰，真是香啊！！！

## 使用switch...case...

  在业务中，我们会有很多场景，不同的场景不同的操作。

  ```js
    function calculatePrice(fruit, price) {
      if (fruit === 'apple') {
        return price;
      } else if(fruit === 'banana') {
        return price * 2;
      } else if(fruit === 'pear') {
        return price * 1.5;
      } else if(fruit === 'tangerine') {
        return price * 0.8;
      }
      .....
    }
  ```

  上面的例子是计算当前水果的价格，但是水果我们肯定不止这几种，有很多。但是如果我们就这样else...if...，那看起来就是长篇的if分支块，可读性和可维护性都不好。
  我们很容易想到使用switch...case...

  ```js
    function calculatePrice(fruit, price) {
      switch(fruit) {
        case 'apple': 
          return price;
        case 'banana':
          return price * 2;
        case 'pear':
          return price * 1.5;
        case 'tangerine':
          return price * 0.8;
        ....
        default: 
          return;
      }
    }
  ```

  这样就代码行数而言，可能也减少不了多少，只是少了很多if...else...分支，看起来更清晰，可读性提高了。但是如果有很多种类，这儿的case也会越来越多，所以我们继续往下看。

## 使用策略（枚举，表驱动编程）

  我们来看看枚举优化

  ```js
    const fruitEnums = {
      apple: 1,
      banana: 2,
      pear: 1.5,
      tangerine: 0.8,
      ......
    }
    function calculatePrice(fruit, price) {
      return fruitEnums[fruit] && fruitEnums[fruit]*price;
    }
  ```

  这样我们添加种类的时候，只需要在枚举中添加就好。
  上面只是举了一个简单的例子，在枚举对象中也可以是函数，比如我们不是简单的做一个计算操作，枚举函数就很有必要了，上面的代码可以扩展为：

  ```js
    const fruitEnums = {
      apple(price) {
        // apple相关价格的逻辑
      },
      banana(price) {
        // banana相关价格的逻辑
      },
      pear(price) {
        // pear相关价格的逻辑
      },
      tangerine(price) {
        // tangerine相关价格的逻辑
      },
      ......
    }
    function calculatePrice(fruit, price) {
      return fruitEnums[fruit] && fruitEnums[fruit]*(price);
    }
  ```

继续上面的例子。如果我们需要添加一种类型，那我们就的在上面枚举类中继续手动添加函数，但是我们更希望说是给出一个添加函数的接口。所以我们就可以继续优化：

  ```js
    const fruitEnums = {
      calculatePrice: {
        apple(price) {
          // apple相关价格的逻辑
          return price;
        },
        banana(price) {
          // banana相关价格的逻辑
          return price*2;
        },
        pear(price) {
          // pear相关价格的逻辑
          return price*1.5;
        },
        tangerine(price) {
          // tangerine相关价格的逻辑
          return price*0.8;
        },
      },
      getPrice: function(type, price) {
        return this.calculatePrice[type] ? this.calculatePrice[type](price) : 0
      },
      setFun: function (type, multiple) {
        this.calculatePrice[type] = function (price) {
          return price*multiple;
        }
      }
    }
    console.log(fruitEnums.getPrice('banana', 12)) // 24
    fruitEnums.setFun('mango', 0.8)
    console.log(fruitEnums.getPrice('mango', 25)) // 20
  ```

  这样，之后再加种类，也可以直接调用接口方法添加，无需手动添加。策略模式是一个值得深究的知识，大家下来可以好好了解一下。

## 合理使用数组方法

* includes
  我们经常会根据后端返回的数据来对界面进行显示控制。像使用vue就会经常见到
  
  ```html
    <div v-if="id===1 || id===2 || id===3" ></div>
  ```

  这儿举得简单得例子，我们就看到界面上有很多||，其实这儿我们像判断得就是当id等于某些值得时候显示。这儿的某些值可能会很多，如果直接在模板上一一列举，那我们的界面看上去就是一长串的v-if
  我们可以将这某些值单独列举出来，然后使用ES6的数组方法includes，判断id是否在数组内。不多说，看代码：

  ```js
    // js中
    const ids = [1,2,3];
    function isInIds(id) {
      return ids.includes(id)
    }
    // 界面判断
    <div v-if="isInIds(id)" ></div>
  ```
  
  这样扩展也很好扩展，万一哪一天产品新增了一个，说id===5的项也显示，这时候我们就只需要修改我们的ids数组，不需要再修改逻辑和模板。其实这儿我觉得和上面的枚举很像，都是将条件单独封装起来，判断时调用。


## 总结

* 单条件优化优先考虑使用操作符，减少或消除if分支。
* 短路条件return提升，减少else分支。
* 嵌套条件深入判断对象使用可选链，减少if条件。
* 互斥条件使用数组方法或者枚举（策略），单独封装，统一接口调用。

> 最后一提，业务中也不是一定越少if越好，适当的if语句，使业务代码更清晰，功能更明确。
