# ProcEvaluate

This ruby gem adds an `evaluate` method to Proc and Object instances through the use of [Refinements][1].

The goal of this gem is to allow evaluation of variables, procs, and lambdas with the same level of flexibility.

Let's say you want to provide an API or DSL where you want the developer to return a value, such as a string, but you want to give the developer \
as much flexibility as possible, so that they can either return a string directly, or return a string from a proc or lambda. \

Your API or DSL may even want to pass in some helpful data to the proc or lambda, which the developer can use to derive their return value.  

But you don't want to write code that handles the annoying differences between procs and lambdas:

```ruby
proc = proc {|a, b, c| [a, b, c] }
proc.call() # [nil,  nil, nil]

lambda = ->(a, b, c) { [a, b, c] }
lambda.call() # ArgumentError: wrong number of arguments (given 0, expected 3)
```

The `evaluate` method has been added to the Object class to return the evaluated value of the variable.
The `evaluate` method is overridden on the Proc class to allow parameters to be passed to lambdas in the same flexible way as procs.
This takes into consideration, required/optional/remaining parameters, and required/optional/remaining keyword parameters.

## Usecase Examples

Your gem or library or dsl requires a value from the developer.
You want to allow the developer the greatest possible flexibility with the value they provide.

A contrived usecase of allowing a developer to define an OpenIdConnect endpoint at config time vs request time:

```ruby
# define endpoint at config time
OpenIdConnectClient.config do |c|
  c.authorize_endpoint 'https://iknowthis.com/at/config/time'
end

# define endpoint at request time
OpenIdConnectClient.config do |c|
  c.authorize_endpoint ->(request) { OpenIdConnectProvider.find_by(name: request.params['provider_name']) }
end
```

Within your code, you can call `evaluate` on the value passed to `authorize_endpoint`.  E.g: `authorize_endpoint_value.evaluate(request)`
* In the case where authorize_endpoint is passed a string, the string is returned
* In the case where authorize_endpoint is passed a Lamda, the lambda is called, and an `OpenIdConnectProvider` is returned

By using the `evaluate` method within your gem/library/dsl, you do not need to be concerned with whether the developer provides a value, proc, or lambda. \
Likewise, the developer has greater flexibility when using you gem/library/dsl as they have greater options for deriving and returning the required value.

## Compatibility

Because this gem makes use of keyword parameters and refinements, it is only compatible with Ruby version 2.0.0 and above (excluding any future release of Ruby 3).

## Usage

In your `Gemfile` add `gem 'proc_evaluate'`.

In your codebase add `require 'proc_evaluate'`.

The refinement methods in the gem can be used by including `using ProcEvaluate` in the file, **class** definition, or **module** definition in which you wish to use the [refinement][1].

### Class and Module usage

The refinement methods can be activated for use within a specific Class or Module.

```ruby
class ProcEvaluateClassExamples
  using ProcEvaluate

  def example1
    a = ->(a) { a }
    a.evaluate('hello', 'world')
  end

  def example2
    a = ->(a, b, c, d, e, f) { [a, b, c, d, e, f] }
    a.evaluate(1, 2, 3, 4)
  end

  def example3
    a = ->(a) { a }
    a.evaluate('Im a proc!!!', 'world')
  end

  def example4
    a = 'im a value!!!'
    a.evaluate('hello', 'world')
  end
end

e = ProcEvaluateClassExamples.new
e.example1 # "hello"
e.example2 # [1, 2, 3, 4, nil, nil, nil]
e.example3 # "Im a proc!!!"
e.example4 # "im a value!!!"

module ProcEvaluateModuleExamples
  using ProcEvaluate
  extend self

  def example1
    a = ->(a) { a }
    a.evaluate('hello', 'world')
  end

  def example2
    a = ->(a, b, c, d, e, f) { [a, b, c, d, e, f] }
    a.evaluate(1, 2, 3, 4)
  end

  def example3
    a = ->(a) { a }
    a.evaluate('Im a proc!!!', 'world')
  end

  def example4
    a = 'im a value!!!'
    a.evaluate('hello', 'world')
  end
end

ProcEvaluateModuleExamples.example1 # "hello"
ProcEvaluateModuleExamples.example2 # [1, 2, 3, 4, nil, nil, nil]
ProcEvaluateModuleExamples.example3 # "Im a proc!!!"
ProcEvaluateModuleExamples.example4 # "im a value!!!"
```

Another example showing a different pattern of usage:

```ruby
class Example
  using ProcEvaluate

  def initialize(value)
    @value = value
  end

  def evaluate_value(*args, **options)
    @value.evaluate(*args, **options)
  end
end

# Example 1: Evaluating a plain value
e1 = Example.new('Hello')
e1.evaluate_value # "Hello"
e1.evaluate_value('World')  # "Hello"
e1.evaluate_value(hello: 'World') # "Hello"

# Example 2: Evaluating a lambda
lambda = ->(req, opt = nil, *rest, keyreq:, keyopt: nil, **options) {
  [req, opt, rest, keyreq, keyopt, options]
}

e2 = Example.new(lambda)
begin
  e2.evaluate_value # ArgumentError: missing keywords: keyreq
rescue => e
  puts e.message
end

e2.evaluate_value(1, keyreq: true) # [1, nil, [], true, nil, {}]
e2.evaluate_value(1, keyreq: true) # [1, nil, [], true, nil, {}]
e2.evaluate_value(1, :optional, keyreq: true) # [1, :optional, [], true, nil, {}]
e2.evaluate_value(1, keyreq: true, keyopt: :optional) # [1, nil, [], true, :optional, {}]
e2.evaluate_value(1, :optional, 'another', 2, keyreq: true, keyopt: :optional, my_key: 'Hello World') # [1, :optional, ["another", 2], true, :optional, {:my_key=>"Hello World"}]
```

### Top Level usage

Please note that the below example will not work if copied and pasted into an irb or pry console.
The `using` statement can only be used at the **top level**, or within a **class** or **module** definition.
To test the example, place the code into a file and run with the command `ruby example.rb`.

```ruby
# example.rb

using ProcEvaluate # activate the refinements for the current file

proc = proc {|a, b, c| [a, b, c] }
proc.evaluate() # [nil,  nil, nil]

lambda = ->(a, b, c) { [a, b, c] }
lambda.evaluate() # [nil,  nil, nil]

var1 = 1
var1.evaluate('hello', 'world') # 1

var2 = ->(a) { a }
var2.evaluate('hello', 'world') # "hello"
```

## Further Reading

For information on Refinements, see:
- https://ruby-doc.org/core-2.0.0/doc/syntax/refinements_rdoc.html
- https://ruby-doc.org/core-2.1.0/doc/syntax/refinements_rdoc.html
- https://ruby-doc.org/core-2.2.0/doc/syntax/refinements_rdoc.html
- https://ruby-doc.org/core-2.3.0/doc/syntax/refinements_rdoc.html
- https://ruby-doc.org/core-2.4.0/doc/syntax/refinements_rdoc.html
- http://yehudakatz.com/2010/11/30/ruby-2-0-refinements-in-practice/

[1]: https://ruby-doc.org/core-2.4.0/doc/syntax/refinements_rdoc.html
