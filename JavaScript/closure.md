作用域闭包

在我早期对于闭包的理解来说，就是当一个函数作为返回值的时候，就形成了一个闭包，但是这个认识还是很模糊，所以我们来真正的认识一下 JS 的闭包。

实质问题

对于闭包的定义，书上是这么写的：当函数可以记住并访问所在的词法作用域时，就产生了闭包，即使函数是在当前词法作用域之外执行。

下面用一个例子来解释一下

    function foo() {
    	var a = 2;
    	function bar() {
            console.log( a ); // 2
    	}
    	bar();
    }
    foo();

这段代码看起来和嵌套作用域中的示例很相似。函数 bar() 可以访问外部作用域中的变量 a 。

但是这个是闭包么？

确切的来说，并不是。这里的 bar() 对 a 的引用的方法是词法作用域的查找规则，而这些规则只是闭包的一部分。

下面我们再来看一段代码

    function foo() {
        var a = 2;
        function bar() {
            console.log( a );
        }
        return bar;
    }
    var baz = foo();
    baz(); // 2

在这里，函数 bar() 的词法作用域能够访问 foo() 的内部作用域，然后我们将 bar() 函数本身当作一个值类型进行传递。在这个例子中，我们将 bar 所引用的函数对象本身当作返回值。

在 foo() 执行后，其返回值（也就是内部的 bar() 函数）赋值给变量 baz 并调用 baz() ，实际上只是通过不同的标识符引用调用了内部的函数 bar() 。

bar() 显然可以被执行，但是在这里，他在自己定义的词法作用域以外的地方执行。

在 foo() 执行后，通常会期待 foo() 的整个内部作用域都被销毁，因为我们知道引擎有垃圾回收器来释放不再使用的内存空间。由于看上去 foo() 的内容不会再被使用，所以很自然的会考虑对其进行回收。

但是闭包可以阻止这个事情的发生，事实上内部作用域依然存在，因此没有被回收。是谁在使用这个内部作用域呢？是 bar() 本身在使用。

由于 bar() 声明的位置，它拥有涵盖 foo() 内部作用域的闭包，使得该作用域能够一直存活，以供 bar() 在之后任何时间里进行引用。

bar() 仍然持有对这个作用域的引用，而这个引用就叫做闭包。

当然，无论使用何种方式对函数类型的值进行传递，当函数在别处被调用时都可以观察到闭包。

    function foo() {
        var a = 2;
        function baz() {
            console.log( a );
        }
        bar(baz);
    }
    function bar(fn) {
    	fn(); // 这就是闭包
    }

把内部函数 baz 传递给 bar ，当调用这个内部函数时，它涵盖的 foo() 内部的作用域的闭包可以观察到了，因为它能够访问 a。

传递函数也可以是间接的。

    var fn;
    function foo() {
        var a = 2;
        function baz() {
            console.log(a);
        }
        fn = baz;
    }
    function bar() {
        fn();
    }
    foo();
    bar(); // 2

无论通过何种手段将内部函数传递到所在的词法作用域以外，它都会持有对原始定义作用域的引用，无论在何处执行这个函数都会使用闭包。

循环和闭包

要说明闭包，for 循环时最常见的例子。

    for( var i = 1; i <= 5; i++) {
        setTimeout( function timer() {
            console.log(i);
        }, i*1000);
    }

这段代码暗示我们会分别输出数字 1 - 5，每秒一次，每次一个。

但是实际上，这段代码在运行时会以每秒一次的频率输出五次6.

首先解释一下 6 是从哪来的。这个循环的终止条件是 i 不再 <= 5。条件首次成立时 i 的值是 6。因此，输出显示的是循环结束时 i 的最终值。

但是为什么会这样呢，为什么会导致它的行为与语义所暗示的不一致？

是因为我们试图假设循环中的每个迭代在运行时都会给自己“捕获”一个 i 的副本。但是根据作用域的工作原理，实际情况是尽管循环中的五个函数是在各个迭代中分别定义的，但是它们都被封闭在一个共享的全局作用域中，因此实际上只有一个 i。

所以缺陷是什么，我们需要更多的闭包作用域，特别是在循环的过程中每个迭代都需要一个闭包作用域。

如果我们通过声明并立即执行一个函数来创建作用域可以么？

    for( var i = 1; i <= 5; i++) {
    	(function() {
    		setTimeout( function timer() {
                console.log( i ); 
    		}, i*1000);
    	})();
    }

这样不行，我们现在很显然的拥有了更多的词法作用域，每个延迟函数都会将 IFEE 在每次迭代中创建的作用域封闭起来。但是如果作用域是空的，那么仅仅对它们进行封闭是不够的，我们的 IFEE 只是一个什么都没有的作用域。它需要有点实质性的内容才能为我们所用。

也就是说它需要有自己的变量，用来迭代存储的 i 的值。

    for(var i = 1; i <= 5; i++) {
    	(function() {
            var j = i;
            setTimeout( function timer() {
    			console.log(j);
    		}, j*1000);
    	})();
    }

这样，他就可以正常工作了。

在迭代中使用 IFEE 会为每个迭代都生成一个新的作用域，使得延迟函数的回调可以将新的作用域封闭在每个迭代内部，每个迭代中都会含有一个具有正确的变量供我们访问。

重返块作用域

我们之前所尝试的解决方法，就是实现我们每次迭代都需要一个块级作用域。之前我们提到了 let 声明，可以用来劫持块作用域，并且在这个块作用域中声明一个变量。

本质上这是将一个块转换成一个可以被关闭的作用域。

因此下面的代码同样可以实现

    for(var i = 1; i <=5; i++) {
        let j = i;
        setTimeout(function timer() {
            console.log(j);
        }, j*1000);
    }

还不够，for 循环头部的 let 声明还有有一种特殊的行为，这个行为指出变量在循环过程中不止被声明一次，每次迭代都会被声明。随后的每个迭代都会使用上一个迭代结束时的值来初始化这个变量。

    for(let i = 1; i <=5; i++) {
    	setTimeout( function timer() {
    		console.log(i);
    	}, i*1000);	
    }

这段代码看起来就很舒服了，所以块作用域和闭包联手，便可以实现很多东西。

模块

还有其他的代码模式利用闭包的强大威力，但是从表面看起来，它们似乎和回调无关。下面来研究其中最强大的一个，模块。

    function CoolModule() {
    	var something = 'cool';
    	var another = [1,2,3];
    	function doSomething() {
            console.log(something);
    	}
    	function doAnother() {
            console.log( another.join("!"));
    	}
    	return {
        	doSomething: doSomething,
        	doAnother: doAnother
        };
    }
    var foo = CoolModule();
    foo.doSomething(); // cool
    foo.doAnother(); 1 ! 2 ! 3

这个模式在 JS 中被称为模块，我们来仔细研究一下这些代码。

首先，CoolModule()只是一个函数，必须要通过调用他来创建一个模块实例。如果不执行外部函数，内部作用域和闭包都无法被实现。

其次， CoolModule() 返回一个用对象字面量语法来表示的对象。这个返回的对象中含有对内部函数而不是内部数据变量的引用，我们保持内部数据变量是私有且隐藏的状态。可以将这个对象类型的返回值看作本质上是模块的公共API。

这个对象类型的返回值被赋值给外部的变量 foo，然后就可以通过它来访问 API 中的属性方法，比如 foo.doSomething()。doSomething() 和 doAnother() 函数具有涵盖模块实例内部作用域的闭包。当通过返回一个含有属性引用的对象的方式来将函数传递到词法作用域外部时，我们已经创建了可以观察和实践闭包的条件。

简单来说，一个模块模式需要具备两个必须条件

- 必须有外部的封闭函数，该函数必须至少被调用一次。
- 封闭函数必须至少返回一个内部函数，这样内部函数才能在私有作用域中形成闭包，并且可以访问或者修改私有状态。

模块模式另一个简单但强大的变化用法是，命名对象将要作为公共 API 返回的对象。

    var foo = (function CoolModule(id) {
        function change() {
    		publicAPI.identify = identify2;
    	}
    	function identify1() {
            console.log(id);
    	}
    	function identify2() {
            console.log(id.toUpperCase);
    	}
    	var publicAPI = {
    		change: change,
    		identify: identify1
    	};
    	return publicAPI;
    })("foo module");
    foo.identify(); // foo module
    foo.change();
    foo.identify(); // FOO MODULE

通过在模块实例的内部保留对公共 API 对象的内部引用，可以从内部对模块实例进行修改，包括添加或删除方法和属性。

未来的模块机制

ES6中为模块增加了一级语法支持。但通过模块系统进行加载时，ES6 会将文件当作独立的模块来处理，每个模块都可以导入其他模块和特定的 API 成员，同样也可以导出自己的 API 成员。

    //bar.js
    
    function hello(who) {
        return "Let me introduce: " + who;
    }
    export hello

    //foo.js
    
    import hello from "bar"
    var hungry = "hippo";
    function awesome() {
        console.log(
        	hello(hungry).toUpperCase();	
        );
    }
    export awesome;

    // baz.js
    
    module foo from "foo"
    module bar from "bar"
    console.log(
    	bar.hello("mike");	
    ); // Let me introduce: mike
    foo.awesome(); LET ME INTRODUCE: HIPPO

import 可以将一个模块中的一个或多个 API 导入到当前作用域中，并分别绑定在一个变量上。 Module 会将整个模块的 API 导入并绑定在一个变量上。 Export 会将当前的模块的一个标识符导出为公共 API 。
