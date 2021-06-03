---
title: C# Dependency Injection in AWS Lambda Functions
date: "2021-06-02T22:40:32.169Z"
template: "post"
draft: false
slug: "c-sharp-dependency-injection-in-aws-lambda-functions"
category: "C#"
tags:
  - "AWS"
  - "Lamnda"
  - "C#"
description: "An approach on how to work with dotnet's DI in AWS Lambda environment."
socialImage: "/media/42-line-bible.jpg"
---

- [The Context](#the-context)
- [Dependency Injection Principle](#dependency-injection-principle)
- [An alternative to DI Approach](#an-alternative-to-di-approach)
- [Conclusion](#conclusion)

An Intro about Dependency injection in Lambda environment

## The Context

When working with .Net core AWS lambda functions, we do not have some fancy `IOC containers` that are able to wire the dependencies in runtime.
If we were working in a "classic" .Net core  project we would have some options of IOC containers, although one largely used in great part of the .Net core projects is the built in IServiceCollection interfaces (IServiceCollection will be used here to refer to .Net core out of the box DI mechanism).
As we are not in a full .Net core environment we do not have access to .Net runtime "injectors", and as we are working with AWS lambdas we do not have a way to hook the DI in runtime (at least not at the time that we write this document).
This context leaves us with some options of non standard way of injecting dependency, and so it leaves this decision to be made.

The first printed books were, at first, perceived as inferior to the handwritten ones. They were smaller and cheaper to produce. Movable type provided the printers with flexibility that allowed them to print books in languages other than Latin. Gill describes the transition to industrialism as something that people needed and wanted. Something similar happened after the first printed books emerged. People wanted books in a language they understood and they wanted books they could take with them. They were hungry for knowledge and printed books satisfied this hunger.



*The 42–Line Bible, printed by Gutenberg.*

But, through this transition, the book lost a large part of its humanity. The machine took over most of the process but craftsmanship was still a part of it. The typefaces were cut manually by the first punch cutters. The paper was made by hand. The illustrations and ornaments were still being hand drawn. These were the remains of the craftsmanship that went almost extinct in the times of Eric Gill.

## Dependency Injection Principle

Short description on Dependency Injection main goals

> In the new computer age the proliferation of typefaces and type manipulations represents a new level of visual pollution threatening our culture. Out of thousands of typefaces, all we need are a few basic ones, and trash the rest.
>
— Massimo Vignelli

Typography is not about typefaces. It’s not about what looks best, it’s about what feels right. What communicates the message best. Typography, in its essence, is about the message. “Typographical design should perform optically what the speaker creates through voice and gesture of his thoughts.”, as El Lissitzky, a famous Russian typographer, put it.

## An alternative to DI Approach

AWS Lambda dependecy injection for dotnet core goes here (code and stuff)

####Example

Config Interface
```csharp
public interface IConfiguration
{
    public ICommonRapidConnectConfig CommonRapidConnectConfig { get; }
    public IPciProxyConfig PciProxyConfig { get; }
}
```

JS Test
```javascript
const my_const = "this is my const";

const my_method = (some_var) => {
  console.log(`This is my var ${some_var}`);
};
```

Python Test
```python 
def do_something(some_var: str):
  print(f'This is my var: {some_var}')

def main():
  my_var = "this is a var"
  do_something(my_var)
```


Typefaces don’t look handmade these days. And that’s all right. They don’t have to. Industrialism took that away from them and we’re fine with it. We’ve traded that part of humanity for a process that produces more books that are easier to read. That can’t be bad. And it isn’t.

> Humane typography will often be comparatively rough and even uncouth; but while a certain uncouthness does not seriously matter in humane works, uncouthness has no excuse whatever in the productions of the machine.
>
> — Eric Gill

We’ve come close to “perfection” in the last five centuries. The letters are crisp and without rough edges. We print our compositions with high–precision printers on a high quality, machine made paper.


*Type through 5 centuries.*

We lost a part of ourselves because of this chase after perfection. We forgot about the craftsmanship along the way. And the worst part is that we don’t care. The transition to the digital age made that clear. We choose typefaces like clueless zombies. There’s no meaning in our work. Type sizes, leading, margins… It’s all just a few clicks or lines of code. The message isn’t important anymore. There’s no more “why” behind the “what”.

## Conclusion

Overall conclusion on the approach
