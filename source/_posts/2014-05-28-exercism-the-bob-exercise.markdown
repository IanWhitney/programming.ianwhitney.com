---
layout: post
title: "Exercism: The Bob Exercise"
date: 2014-05-28 21:14:52 -0500
comments: true
categories: 
---

[The exercise's readme](https://github.com/IanWhitney/exercism_problems/tree/master/ruby/bob)

[The exercise's test suite](https://github.com/IanWhitney/exercism_problems/blob/master/ruby/bob/bob_test.rb)

This is the first exercise you'll get if you're working on Ruby in [Exercism](http://exercism.io/). And now's probably a good a time as any to figure out what you want to do with Exercism. Are you using the exercises as a way to learn the language? Or are you trying to improve something else?

With Ruby, I use [Exercism](http://exercism.io/) as a way to experiment with designs. I'm pretty comfortable with the language (though I learn new things every day), so my interest is more focused on improving the expressiveness and maintainability of my code. That's my personal focus. It may not be yours. But this interest is what is going to drive these nitpicking posts, so I should just be clear about what my coding goals were.

When working on these exercises, I've found a pretty good approach and it's one I learned from Exercism's creator, Katrina Owen: Get the tests passing as quickly as possible, then refactor while <strong>never</strong> letting the tests fail again. That first pass is going to be terrible code, and that's fine. Its main goal is to get your tests passing. With that safety net in place, you can start to fix the most egregious mistakes. Keep chipping away at the parts of the code you dislke. Eventually you'll find that you like it. And, bonus, your tests will still pass.

I'd like to include specific Git commits in these posts, but for the Bob exercise I didn't commit until the very end. So I'll have to recurstruct.

My initial solution, like many on Exercism, started off like this:

```ruby
class Bob
  def hey(statement)
    if statement.strip.empty?
      "Fine. Be that way!"
    elsif statement.upcase == statement && statement.downcase != statement
      "Woah, chill out!"
    elsif statement.end_with?("?")
      "Sure."
    else
      "Whatever."
    end
  end
end
```

Some people use more regex, while I like to stick to sugar methods like `end_with` (which I learned about thanks to this exercise!). Whichever. The end point is the same, you have an if statement that handles your 4 repsonses. Tests passing and we got here pretty quickly. 

This is where most people on Exercism stop. That might be because they don't get nitpicks, or because they like their solutions. But since I want to nitpick my designs and try new approaches, it's time to start changing the code I don't like.

My first target is the conditionals. What is special about `statement.strip.empty?`?. It doesn't communicate its meaning.

### Refactoring 1

```ruby
class Bob
  def hey(statement)
    if silence?(statement)
      "Fine. Be that way!"
    elsif yelling?(statement)
      "Woah, chill out!"
    elsif question?(statement)
      "Sure."
    else
      "Whatever."
    end
  end

  def silence?(statement)
    statement.strip.empty?
  end

  def yelling?(statement)
    statement.upcase == statement && statement.downcase != statement
  end

  def question?(statement)
    statement.end_with?("?")
  end
end
```

And now I know why Bob is responding "Woah, chill out!". It's because someone was yelling.

But who is yelling? The way these methods are written, it looks like Bob is yelling. `Bob.new.yelling?` is totally viable code, and that code makes it look like Bob is a yeller. Also, we have 3 branches of our if/else statement that are all weirdly similar. This looks to me like an unecessary responsibility has been given to Bob. Bob needs to respond to different types of statements.

### Refactoring 2

```ruby
class Bob
  def hey(statement)
    statement = StatementParser.parse(statement)
    if statement.silence?
      "Fine. Be that way!"
    elsif statement.yelling?
      "Woah, chill out!"
    elsif statement.question?
      "Sure."
    else
      "Whatever."
    end
  end
end

class StatementParser
  def self.parse(statement)
    self.new(statement).parse
  end

  def initialize(statement)
    self.statement = statement
  end

  def parse
    if silence?
      Silence.new
    elsif yelling?
      Yelling.new
    elsif question?
      Question.new
    else
      Statement.new
    end
  end

  private

  def silence?
    statement.strip.empty?
  end

  def yelling?
    statement.upcase == statement && statement.downcase != statement
  end

  def question?
    statement.end_with?("?")
  end

  attr_accessor :statement
end

class Statement
  def silence?
    false
  end

  def yelling?
    false
  end

  def question?
    false
  end
end

class Question < Statement
  def question?
    true
  end
end

class Yelling < Statement
  def yelling?
    true
  end
end

class Silence < Statement
  def silence?
    true
  end
end
```

And that's...quite a bit bigger. But we have 3 distinct sets of responsibilities now. There's Bob, who responds to statements. There's StatementParser that figures out what a statment is. And there's a small set of Statement and its children that answer expressive questions about their nature.

There are some downsides here. You can't really have Statements that have two roles. Like, you couldn't have a Yelling Question. Given the test suite, that doesn't seem to be a problem, though. But it is a trade-off to be aware of. And you can't have Bob be anything but a surly teenager. Again, the test suite doesn't indicate that you'll ever want Bob to give a response beyond these 4. But imagine that this is the next step. Bob has grown up and now gives the responses of a college student, not a teenager.

### Refactoring 3

```ruby
class Bob
  attr_accessor :response
  def initialize(response = TeenagerResponse)
    self.response = response
  end

  def hey(query)
    response.to(StatementParser.parse(query))
  end
end

class Response
  def self.to(statement)
    self.new(statement).say
  end

  def initialize(statement)
    self.statement = statement
  end

  def say
    self.send("to_#{statement.class.to_s.downcase}".to_sym)
  end

  def method_missing(method_name, *arguments, &block)
    if method_name.to_s =~ /to_(.*)/
      to_everything
    else
      super
    end
  end

  private

  attr_accessor :statement
end

class TeenagerResponse < Response
  private

  def to_silence
    "Fine. Be that way!"
  end

  def to_yelling
    "Woah, chill out!"
  end

  def to_question
    "Sure."
  end

  def to_everything
    "Whatever."
  end
end

class CollegeResponse < Response
  def to_silence
    "Broseph! Don't leave me hanging!"
  end

  def to_yelling
    "Bro!"
  end

  def to_question
    "Bro?"
  end

  def to_everything
    "Bro."
  end
end

class StatementParser
  def self.parse(statement)
    self.new(statement).parse
  end

  def initialize(statement)
    self.statement = statement
  end

  def parse
    if silence?
      Silence.new
    elsif yelling?
      Yelling.new
    elsif question?
      Question.new
    else
      NullStatement.new
    end
  end

  private

  def silence?
    statement.strip.empty?
  end

  def yelling?
    statement.upcase == statement && statement.downcase != statement
  end

  def question?
    statement.end_with?("?")
  end

  attr_accessor :statement
end

class NullStatement
end

class Question
end

class Yelling
end

class Silence
end
```

I work at a university, so I can vouch for the total accuracy of those responses. 

And, wow, that's a lot of code. A couple things to note between the last version and this one. We've taken the responsibility of responding and put it into Response objects. When we create an instance of Bob, we can set what kind of responses Bob will give. Teenager is still the default as no one ever fully outgrows those teenage years.

Also, I got a little goofy with the metaprogramming. I never like it when I reach for the `method_missing` stick, but it felt OK here. If a set of responses can't handle the statement, it falls back to its "to_everything" method. A big downside here is that if I create a set of responses but don't imlement `to_everything` then stuff starts breaking.

Also worrying are the 4 empty classes. I did this so that I could do a dynamic message sending in the `say` method. But looking at it now, I think it's cleverness for its own sake. But I don't have a good replacement in mind, so let's just leave that as is.

### Summary

The initial implementation was 12 lines of code. The last solution is 123 lines containing 9 classes. One goal of Exercism is to "practice writing expressive code". Did I do that here? I'm not sure. I have a solution that is obviously more flexible. And I think the responsibilities in my final code are more clear. But there's no doubt that tracing through the code is harder.

This is in no way the 'right' solution. Or, possibly, even a 'good' solution. But it is a solution. And it's an example of the way you can use the problems in Exercism to push your code in suprising directions.
