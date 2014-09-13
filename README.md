Gremlinpy
=========

Gremlinpy is a small library that allows you to write pure Python and output Gremlin script complete with bound parameters that can be run against a Tinkerpop Gremlin server instance. It is meant to be a low-level way for your application to communicate its intent with the graph server.

##Setup

    python setup.py install

##Dependencies
None

##Overview

Python's syntax nearly mirrors Groovy's one-to-one so Grelinpy allows for an easy to manipulate Python object that will produce a Gremlin string as a result.

Gremlinpy works by tokenizing every action taken on the `gremlinpy.gremlin.Gremlin` instance into a simple linked list. Each `Gremlin` instance starts with a `GraphVariable` and chains the rest of the tokens to it.

If you wanted to produce a Gremlin script like this:

    g.v(12).outE('knows').inV

You would simply write:

    g = Gremlin()
    g.v(12).outE('knows').inV
    
Once that is converted to a string, your gremlin instance will hold the bound params (you have the ability to control the names of the bindings).

    script = str(g) #g.v(GP_KKEI_1).outE(GP_KKEI_2).inV
    params = g.bound_params #{'GP_KKEI_1': 12, 'GP_KKEI_2': 'knows'}

It's that simple, but Gremlinpy allows you to write very complex Gremlin syntax in pure Python. And if you're having trouble expressing the Gremlin/Groovy in the Python model, it allows for straight string manipulation complete with parameter binding.

Though the usage of Python's magic methods, we can compose a Gremlin string by recording every attribute, item, or call against the instance and creating a token for it. There are eight token types that allow you to express Gremlin with Python: 

__GraphVariable__: This is the root token in the list. It is always present and will be used in every Gremlin string produced. It can be nullified by calling `Gremlin.set_graph_variable('')`. That is useful for when you are running scripts that may not interact with the graph or when your graph variable is something other than the letter "g".

*__Attribute__: Attributes are things on the Gremlin chain that are not functions, arent closures, and are not indexes. They stand alone and are defined when you call any sequence without parenthesis on your Gremlinpy instance. 
     
    g.a.b #g.a.b -- a and b are the attribues    
     
*__Function__: Functions are called when you add parenthesis after an attribute. The function object will assume that the last argument passed to the function is the one that will be bound.

    g.V(12) #g.v(GP_UUID_1)
    g.bound_params # {'GP_UUID_1': 12}
    
    g.v.has("'name'", 'T.eq', 'mark') #g.v.has('name', T.eq, GP_CXZ_1)
    g.bound_params # {'GP_CXG_1': 'mark'}
    
> Take a closer look at the second example as there are two things going on: the last parameter passed to the has function is automaticaly bound and if you want the value to be quoted in the resulting Gremlin string, you __MUST__ double quote it in Python. 

*__UnboundFunction__: This allows you to call a function, but not have the instance automatically bind any of the params. It has a differnt syntax than just chaning a function in the previous exmaple:

    g.unbound('function', 'arg', 1, 3, 5) #g.function(arg, 1, 3, 5)

*__Index__: Indexes are done in Groovy in a way that is not directly allowed in Python's syntax. However, Python's slices should convert over pretty easily:

    g.function('arg')[1:50] # g.function(GP_XXX_1)[1..50]
    
    g.x[1] # g.x[1]

*__Closure__: Closures simply allow you to put things between curly braces. Since it is an error to add curly braces to the end of Python objects, this has its own method on the `Gremlin` instance:

    g.func.close('im closing this') # g.func{im closing this}

*__ClosureArguments__: Groovy allows for inline lambda functions in a syntax that isn't supported by Python. To define the signature for the closure you simply pass in args after the first argument on a closure:

    g.func.close('body', 'x', 'y') # g.func{x, y -> body }
    
*__Raw__: Raw allows you to put anything in and have it passed out the same way. It doesnt put anything before or after the call. It is useful for when you're doing something that cannot easily map:

	g.set_graph_variable('')
		.raw('if(').raw('1 == 2').raw(')')
		.close("'never'")
		.else.close("'always'")
	# if(1 == 2){'never'}else{'always'}
	
> note: this is just an example, there are better ways to do complex composition

> \* -- these methods have memebers that are reserved words

###Overloading

The `Gremlin` instance has memebers that are basically reserved words and will not be passed to your resulting gremlin script. 

These include:

* \__init__
* reset
* \__getitem__
* set_graph_variable
* any other mehtod or property on the object

If you need the resulting gremlin script to print out '\__init__' or one of the reserved words, you can simply call `add_token` on your instance:

    init = Function(g, '__init__', ['arg'])
    g.add_token(init) # g.__init__(GP_XXX_1)
    
    init = Attribute(g, '__init__')
    g.add_token(init).xxx() # g.__init__.xxx()
    
    add_token = UnboudFunction(g, 'add_token', [5, 6])
    g.add_token(add_token) # g.add_token(5, 6)
    
###Binding Params
The last parameter passed in a function is automatically bound. Each `Gremlin` instance creates a unique key to hold the bound parameter values to. However, you can manually bind the param and pass a name that you deisre.

    bound = g.bind_param('my_value', 'MY_PARAM')
    
    g.v(bound[0]) # g.v(MY_PARAM)
    g.bound_params # {'MY_PARAM': 'my_value'}

###Nesting Instances

Gremlinpy gets interesting when you want to compose a very complex string. It will allow you to nest `Gremlin` instances passing any bound params up to the root instance. 

Nesting allows you to have more control over query creation, it offers some sanity when dealing with huge strings.

    g = Gremlin()
    i = Gremlin()
    
    i.set_grap_variable('').it.setProperty('age', 33)
    g.v(12).close(i) # g.v(GP_XXQ_1){it.setProperty('age', GP_UYI_1)}
    
    g.bound_params # {'GP_XXQ_1': 12, 'GP_UYI': 33}


##Statements
Coming soon

##Performance Tweaks
###Always Manually Bind Params
If your Gremlin server instance has query caching turned on, manually binding params will allow you to create statements on the server that will  pre-parse your query the second time you run it an return results quicker.

If you are not manually binding params, everytime you call a script, even the same script, a differnt one is being sent to the server.

    g.v(12) # g.v(GP_XSX_1)
   
    #later 
    g.v(12) # g.v(GP_POI_1)
   
If you manually bind the param with a name, the same script will be sent to the server and it will drastically cut down on execution times. This is true even if the param values are changed:

    id = g.bind_param(12, 'eyed')
    g.v('eyed') #g.v(eye_d)
    
    id = g.bind_param(9999, 'eyed')
    g.v('eyed') #g.v(eye_d)  <--- this one executes faster than the first