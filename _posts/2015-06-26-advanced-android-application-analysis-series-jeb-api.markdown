---
published: true
title: Advanced Android Application Analysis Series – JEB API
layout: post
---
Author:[Flanker](https://twitter.com/flanker_hqd)

    Drops is one of the greatest platforms for security-related technical blogs in China. We dedicate to translate every amazing post into English and push it to this website, meanwhile we would be very honored if you could submit your atricles at drops@wooyun.org.

translated from:http://drops.wooyun.org/mobile/6665

#0x00 Introduction&Background

---------------

Are you still using smali and jugui to grep grep and grep?  This series focusing on Soot and JEB will explain the advanced analysis of Android applications. Just use your new powerful weapons.

JEB is the de facto standard for the static analysis of Android applications. Apart from its accurate decompiling results and high tolerance, JEB API also facilitates our writing plugins to process the source files, the implementation of de-obfuscation and even some more advanced applications of analysis to promote subsequent analysis. The starting pages of this series will discuss the usage of JEBAPI and implement it in real life to perform de-obfuscation by using the traces that developers left behind. Now, let's check out the plugin we write, which can perform automatical de-obfuscation.

Without de-obfuscation:

![image](https://quip.com/blob/JDMAAAXinpL/w8sVke0ib3wv3Gt10YcNOQ?s=GkVwA30X5Se4)

![image](https://quip.com/blob/JDMAAAXinpL/w6wVJXMeqAn78KCZgKnxIg?s=GkVwA30X5Se4)

With de-obfuscation:

![image](https://quip.com/blob/JDMAAAXinpL/2cQcM2m2yf1jmqa4I5QoIw?s=GkVwA30X5Se4)

![image](https://quip.com/blob/JDMAAAXinpL/t-2uJr35_1bJPkxJ9gYlNQ?s=GkVwA30X5Se4)

As you can see, the class names and field names has been recovered. I bet you must wonder how could this happen. We'll begin with the constructure of JEBAPI:

#0x01 JEB AST API structures

----------------

JEB AST is similar to Java AST. But the difference is that JEB AST is simpler. All AST Element's implementation of jeb.api.ast.IElement is either inherited from jeb.api.ast.NonStatement, or from jeb.api.ast.Statement. Their relationship is as shown in the following figure :

![image](https://quip.com/blob/JDMAAAXinpL/guh2lCt8dl1kDwh92plkug?s=GkVwA30X5Se4)

The basic `IElement` interface defines `getSubElements` abstract method, however its different concrete subclasses have different implementations and return values. For example, the `Method` subclass' implementation of `getSubElements` returns the method function's parameter definition statements and the method function's body block statements. While on the other hand, the `IfStmt` subclass' implementation of `getSubElements` will return the `Predicate` instance in branching statement (i.e. if(*Predicate*)) and the remaining every statement blocks of branch bodies. An `Assignment` subclass's subElements are it's left and right operators and the `Operator`, in their respective orders. As the actually implementation of `getSubElements` varies depending on calling instance type, when we write our own scripts, we normally would not use this function directly. Instead, we will use more specifically functions provided by each subclass, for example, the `Assignment` subclass provides `getLeft` and `getRight` function for us to retrive the left and right operator in an assign statements.

In the case of following functions, we will analyze its composing AST elements.

    boolean isZtz162(Ztz ztz) {
        boolean bool = true;
        Redrain redrain = Redrain.getInstance("AnAn");
        if(redrain.canShoot()) {
            redrain.shoot(163);
            if(ztz.isDead()) {
                bool = false;
            }
        }
        else if(ztz.height + Integer.parseInt(ztz.shoe) > 162) {
            bool = false;
        }
    
        return bool;
    }

First, let's look at theNonStatement

**NonStatement**

As described in the document, a NonStatement is the Base class for AST elements that do not represent Statements. That's to say,  every AST structure but a Statement is  inherited from NonStatement shown in the following figure:

![image](https://quip.com/blob/JDMAAAXinpL/-7zI5BbQl_ChnDJgkRA3Xw?s=GkVwA30X5Se4)

Different from the  Expression, a NonStatement contains some of the advanced structures, such as jeb.api.ast.Classand jeb.api.ast.Method.  The two won't appear in the AST structure of the statement. They respectively represent the Class structure and the Method structure. Note: Don't confuse them with the Class and Method used in reflection statement.

**Statement**

A Statement by definition represents a*statement*, but it is worth noting that the *statement* here does not represent a single statement. A Statement inherited from the Compoundmay also contain otherStatement. For example, the following code:

![image](https://quip.com/blob/JDMAAAXinpL/k46nEUxyVJb1OTYQPLOU-A?s=GkVwA30X5Se4)

This is in fact an IfStm  completely inherited from Compound, which is a Statement.

The inheritance graph of Statement is shown in the following figure.

![image](https://quip.com/blob/JDMAAAXinpL/TGoR_nIS1H7E_Dg8fX_3ug?s=GkVwA30X5Se4)

Non-Compound Statement is the most basic sentence structure, the child nodes of which can only be constructed by Expression without any block. For instance, Assignment can get ILeftExpression and IExpression by calling getLeft and getRight, namely ILeftExpression and IExpression. ILeftExpression represents the Expression on the left, such as variables. But apparently constants do not implement ILeftExpression interface.

**Compound**

A Compound represents the syntax block collection of multiple statements collections. Each syntax block is presented as a Block(the subclass of Compound) through calling getBlocks to get it. All branching statements are inherited from Compound, as shown in the following figure:

In the above example, IfStmt is a Compound, we use getBranchPredicate (idx)to get Predict which is the Expression-ztz.isDead (). This Expression is actually a subclass Call. We can use getBranchBody (idx)to get the Block in if and if-else, and use getDefaultBlock to get the Block in else.

**IExpression**

A IExpression represents the most basic AST nodes. The implementation relationship is  as follows:

http://static.wooyun.org//drops/20150625/2015062502395525654.jpg

The Expression class which implements the IExpression interface represents the statement pieces for arithmetic and logical operations, such as a+b, "162" + ztz.toString (),!ztz, redrain* (ztz-162), and so on. Besides, Predicate class is the direct subclass of the Expression. For example, in the statement If ( Ztz162), the left value of Predicate is ztz162, , the right value of this identifier is null.

In the case of ztz.test(1) + ”height" + 162, the structure and node type of this Expression are as follows:

![image](https://quip.com/blob/JDMAAAXinpL/tlBCiWIbni3IxSJzYj9ZzQ?s=GkVwA30X5Se4)

Note the following points:

* The structure of the Expression is from right to left
* Call does not provide API to get the caller, but you can use getSubElements() to get the caller. The order of return is :
    * callee method
    * calling instance (if instance call)
    * calling arguments, one by one

InstanceField, StaticField and Field

Their relations are shown in the following figure:

![image](https://quip.com/blob/JDMAAAXinpL/guh2lCt8dl1kDwh92plkug?s=GkVwA30X5Se4)

Both InstanceField and StaticField contain the Field. The InstanceField calls getInstance to get an IExpression which is the container of Field. The Field itself is an element of Class, while InstanceField and StaticField is the specific instantiation of it.

**Analyzing Method cases**

The AST structure of function Ztz162 is as follows:

    - jeb.api.ast.Method  (getName() == "isZtz162") => getBody()
        - Block => block.get(i) //traverses the statements in the block
            - Assignment "boolean bool = true"  => getSubElements
                - Definition "boolean bool"
                    - Identifier "bool"
                - Constant "true"
            - Assignment "Redrain redrain = Redrain.getInstance("AnAn");"  => getSubElements
                - Definition => getSubElements (note that it is the returned results of getLeft of  Parent assignment (left-value))
                    - Identifier "redrain"
                - Call "Redrain.getInstance("AnAn)"" (note that it is the returned results of getRight of Parent assignment (right-values))
                    - ...(omit)
            - IfStmt (Compound) => getBlocks()
                - Block (if block) => block.get(i)  traversing the statements in the block
                    - Call "redrain.shoot(163);"
                    - IfStmt (Compound) 
                        - ...omit
                - Block (elseif block) => block.get(i) Traverse the statements in the block
                    - Assignment "bool = false'"
                    - ..omit

That's all of my analysis for AST structure. In this post, we have explained some the most typical cases. Additionally JEB provides theJeb.API.Dex, and the operational API for dex files. As there are a lot of materials regardig this, we won't discuss it here.

You can use the following code to recursively print every Element in a  Method:

    class test(IScript):
    
    def run(self, j):
        self.instance = j
        sig = self.instance.getUI().getView(View.Type.JAVA).getCodePosition().getSignature()
        currentMethod = self.instance.getDecompiledMethodTree(sig)
        self.instance.print("scanning method: " + currentMethod.getSignature())
    
        body = currentMethod.getBody()
        self.instance.print(repr(body))
        for i in range(body.size()):
            self.viewElement(body.get(i),1)
    
    def viewElement(self, element, depth):
        self.instance.print("    "*depth+repr(element))
        for sub in element.getSubElements():
            self.viewElement(sub, depth+1)

The output results are as follows:

    jeb.api.ast.Block@5909b311
        jeb.api.ast.Assignment@bcb4ec2
            jeb.api.ast.Definition@66afd874
                jeb.api.ast.Identifier@38ffa6bd
            jeb.api.ast.Constant@181bdf87
        jeb.api.ast.Assignment@4df0246e
            jeb.api.ast.Definition@50e7d9bb
                jeb.api.ast.Identifier@2587ad7c
            jeb.api.ast.Call@6e8ebb23
                jeb.api.ast.Method@5ca02f89
                    jeb.api.ast.Definition@1890fae1
                        jeb.api.ast.Identifier@5646d660
                    jeb.api.ast.Block@44a464e0
                jeb.api.ast.Constant@4dad155
        jeb.api.ast.IfStm@298ea172
            jeb.api.ast.Predicate@530958ae
                jeb.api.ast.Call@a9d3219
                    jeb.api.ast.Method@56440cc0
                        jeb.api.ast.Definition@da13d7f
                            jeb.api.ast.Identifier@54cc63d6
                        jeb.api.ast.Block@36aea218
                    jeb.api.ast.Identifier@2587ad7c
            jeb.api.ast.Predicate@313f1b4
                jeb.api.ast.Expression@12616200
                    jeb.api.ast.InstanceField@3768f76d
                        jeb.api.ast.Identifier@4c4c3186
                        jeb.api.ast.Field@198ed96b
                    jeb.api.ast.Call@71640ce8
                        jeb.api.ast.Method@5f8b8d80
                        jeb.api.ast.InstanceField@42f6ff81
                            jeb.api.ast.Identifier@4c4c3186
                            jeb.api.ast.Field@6600907f
                jeb.api.ast.Constant@2f0eb62a
            jeb.api.ast.Block@6ed99788
                jeb.api.ast.Call@f6b9a93
                    jeb.api.ast.Method@617130cd
                        jeb.api.ast.Definition@4e3b14b5
                            jeb.api.ast.Identifier@8cc9f33
                        jeb.api.ast.Definition@31e7d1c8
                            jeb.api.ast.Identifier@6a7dbb10
                        jeb.api.ast.Block@64844e0e
                    jeb.api.ast.Identifier@2587ad7c
                    jeb.api.ast.Constant@2a20acb0
                jeb.api.ast.IfStm@47296c6b
                    jeb.api.ast.Predicate@708d094c
                        jeb.api.ast.Call@3b5d964e
                            jeb.api.ast.Method@7d36f954
                                jeb.api.ast.Definition@242b3a05
                                    jeb.api.ast.Identifier@11ee30d0
                                jeb.api.ast.Block@2cc6b0e2
                            jeb.api.ast.Identifier@4c4c3186
                    jeb.api.ast.Block@2886dc65
                        jeb.api.ast.Assignment@2def7fac
                            jeb.api.ast.Identifier@38ffa6bd
                            jeb.api.ast.Constant@46a70cc3
            jeb.api.ast.Block@136fa72
                jeb.api.ast.Assignment@407452fd
                    jeb.api.ast.Identifier@38ffa6bd
                    jeb.api.ast.Constant@46a70cc3
        jeb.api.ast.Return@14f4811a
            jeb.api.ast.Identifier@38ffa6bd
    

#0x02 Case Analysis -development environment configuration

-----------------

JEB natively supports Java and Python for development, the latter of which is achieved by Jython. To simplify our work, all of our cases use Python. Personally, I suggest you use Scala if you choose java, because java itself is too complex.

**Java**

In eclipse, you need to configure the library path in classpath. Set the library path to point to bin/jeb.jar and set javadoc path to point to jeb/doc/apidoc.zip.

![image](https://quip.com/blob/JDMAAAXinpL/V9_hSjWgPm1upfxbNeETYw?s=GkVwA30X5Se4)

![image](https://quip.com/blob/JDMAAAXinpL/S34r_koyHdyW8NNy1MCVeQ?s=GkVwA30X5Se4)

**Python**

You will take some time to configure Python, because JEB doesn't provide corresponding skeleton. As a result, Python lacks code completion in IDE by default. Therefore, you have to manually configure Python. We use PyCharm and one of its plugin JythonHelper, which can generate a skeleton to provide basic code completion.

![image](https://quip.com/blob/JDMAAAXinpL/6OOVeqf0t9xr_3QJ0elQJw?s=GkVwA30X5Se4)

![image](https://quip.com/blob/JDMAAAXinpL/GBXvu21B6iZ5b3rRRfa9Ng?s=GkVwA30X5Se4)

After you have configured environment, we'll write a simple plugin:print the method signature which the  cursor is at, the code is like this:

    from jeb.api import IScript
    from jeb.api.ui import View
    class test(IScript):
    
        def run(self, j):
            self.instance = j
            sig = self.instance.getUI().getView(View.Type.JAVA).getCodePosition().getSignature()
            currentMethod = self.instance.getDecompiledMethodTree(sig)
            self.instance.print("scanning method: " + currentMethod.getSignature())

Save as test.py, click File->Run Script->test.py, and JEB will print the signature of the function which the  cursor is at in the following console.

#0x03 Summary

-----------------

This article introduces the basics of JEB Java AST API and plugins writing. It is also complement for an APIDoc reference. In the next article we will  show you how to write advanced plugins based on cases.

Source code and test sample can be found at https://github.com/flankerhqd/jebPlugins.
