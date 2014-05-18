---
layout: post
title: "Combining GADTs and Polymorphic Variants"
description: ""
category: Ocaml
tags: []
---
{% include JB/setup %}

In Ocaml, GADTs are an efficient mechanism to describe types in order to make
programs safer. Yet in some cases, it is hard to generalize functions over
different signatures of a same type: for a type `t` which has two instances,
such as `int t` and `char t`, if one wants to write a type that handles both
signatures, this must be done through the use of a polymorphic type variable
`'a`, for instance, `type 'a container = T of 'a t`. While in most of the
cases, this behavior is fine, this can become a problematic when we want a
type without a polymorphic variable that is able to generalize different
signatures of a same type.

In this post, we will see an easy way to keep the strength of GADTs, while
still being able to generalize functions over them. We first introduce a simple
example which uses a generic type in order to highlight our thought process,
and then show how polymorphic variants can be used to refine this generic type.
Then, we try to use GADTs to refine the same type and show the problem that
appears when using them, to finally show how polymorphic
variants and GADTs can be combined in order to be able to abstract over
different type signatures.

While I believe this trick is widely known by Ocaml experts, I did not find any
article which presented this technique. Another thing is that, although I use
GADTs, phantom types could be used to achieve the same result.

## An initial tentative

My original problem is the following: I have an AST representing constructs of
a language that is made of several elements, some of them referencing
`lang_types`, a type which describes the types of my language (the list being
used to describe the sequence of namespaces that leads to the final symbol).
The following example
shows a simplified version of this AST:

{% highlight ocaml linenos %}
type lang_type = string list

type symbol = string
type name = string

type ast =
  | Namespace of symbol * ast list
  | Attribute of name * lang_type
  (* name_of_type * parent_type *)
  | Type of name * lang_type
{% endhighlight %}

In this example, the type `ast` describes 3 constructs of our language:

* __Namespaces__, which are made of a name, their contents being a list of other
  declarations.
* __Attributes__ which contain a name and a type (such as `("Foo", ["int"])`
* __Type declarations__, which are made of a name and of a parent type (other
  contents being omitted).

In this context, the type `ast` appears to have a simple design which should
work for most of the cases. Yet, I also want to add support for pointers in my
language. One way easily extend our ast could be to refine the type `lang_type`
in order to support pointers. Here `lang_type` is modified to have a constructor
that describes pointers, while another describe non-pointer types, that is,
direct types.

{% highlight ocaml linenos %}

type lang_type =
  (* name * constness *)
  | Ptr of string list * bool
  | Direct of string list

type ast =
  | Namespace of symbol of ast list
  | Attribute of name * lang_type
  (* name_of_type * parent_type *)
  | Type of name * direct lang_type

{% endhighlight %}

But with this approach, problems appear: the way `lang_type` is used is now
"too broad" and our AST accepts constructs that are invalid. For instance, it
is now possible to declare a type that extends a pointer, which makes no sense.
In order to solve this problem, there are two approaches: either split the
types, using two different types to describe either pointers or direct values,
or using GADTs, it would be possible to define a `ptr lang_type` and `direct
lang_type`, in order to differentiate the use cases in `ast`.

Splitting type works and it is easy to setup: if we use polymorphic variants to
describe `lang_type`, it is quite easy to refine `lang_type` and to specialize
the different elements of `ast`:

{% highlight ocaml linenos %}
type ptr = [`Ptr of string * bool]
type direct = [`Direct of string]

type any = [ptr | direct]

type ast =
  | Namespace of symbol of ast list
  | Attribute of name * any
  (* name_of_type * parent_type *)
  | Type of name * direct

{% endhighlight %}

Here, we can interchangeably use `ptr`, `direct` or `any` to describe types in
our language. `Attribute` accepts any type, while `Type` only accepts `direct`
types. One problem of this approach is that `ptr` and `direct` are totally
independent, and conceptually, it would be good to have them under a common
definition. One way to do this could be to use GADTs.

## It's better with GADTs !

Using GADTs, we can now refine our type `lang_type` so that the different cases
can be distinguished:

{% highlight ocaml linenos %}
type ptr
type direct

type 'a lang_type =
  | Ptr : string list * bool -> ptr lang_type
  | Direct : string list -> direct lang_type

{% endhighlight %}

Here, when the constructor `Ptr` is called, the type returned will be of `ptr
lang_type`, while when calling `Direct`, we'll have `direct lang_type`.
The main interest in this approach is that we can distinguish between different
kinds of types, and ensure that our types are efficiently described. The
following example show how types of our Ast can now be refined to efficiently
describe types of our language: the `'a` replacing the `any` of our example
that uses polymorphic variants. `Attribute` now accepts any kind of
`lang_type`, and `Type` uses `direct lang_type`.

{% highlight ocaml linenos %}
  | Attribute of name * 'a lang_type
  (* name_of_type * parent_type *)
  | Type of name * direct lang_type
{% endhighlight %}

But the rest of our ast definition is problematic:

{% highlight ocaml linenos %}
type 'a ast =
  | Namespace of symbol of 'a ast list
  | Attribute of name * 'a lang_type
  (* name_of_type * parent_type *)
  | Type of name * direct lang_type
{% endhighlight %}

Because we need an `'a` type to describe the contents of `Attribute`, the type
`ast` must now be defined as being parameterized by this polymorphic type
variable. The consequence is that `Namespace` must contain an `'a ast list`,
which is completely incorrect, as it forces the type of elements of our
namespace to be the same: either a pointer or a direct type. What we'd need is
the `any` type of our example with polymorphic variants, which facilitates the
typing of `ast` by restricting the polymorphic variable to our two use cases,
`ptr` and `direct`.

## Adding polymorphic variants

The solution is to use polymorphic variants to describe the different kinds of
`lang_type` that can be met, and more interestingly, to be able to generalize
over them:

{% highlight ocaml linenos %}
type ptr = [`Ptr]
type direct = [`Direct]
type any = [ ptr | direct ]

type 'a lang_type =
  | Ptr : string list * bool -> [> ptr] lang_type
  | Direct : string list -> [> direct] lang_type

type ast =
  | Namespace of symbol of ast list
  | Attribute of name * any lang_type
  (* name_of_type * parent_type *)
  | Type of name * direct lang_type

{% endhighlight %}

In this example, the type `any` is the union of our two previously defined
types `ptr` and `direct`. The type `lang_type` is also rewritten, so that `Ptr`
and `Direct` return a more general signature: the syntax `[> ptr] lang_type`
tells Ocaml that the type being returned contains at least `Ptr`. In our use
case, it allows a value constructed with `Ptr` to be either a `ptr lang_type`
or a `any lang_type`. This means that `any lang_type` encompasses the kinds of
type we can create with `lang_type`, and it hides the `'a` type variable that
would be used otherwise.

With this new definition, we are able to refine the type of our AST:
`Attribute` is now made of `any lang_type`, which means that it can contain
either pointers or direct types.

## Conclusion

In this post, we saw a way to use GADTs to describe different specializations
of a same type. The use of polymorphic variants allows to perform the union of
those specializations in order to avoid polymorphic type variables, which in
our use case appeared to be problematic.
