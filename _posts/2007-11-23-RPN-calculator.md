---
layout: post
title: "RPN calculator"
date: 2007-11-23
---

I had some time to spend waiting for my flight home a few days ago, so I decided to write a calculator using reverse polish notation. 
See <http://en.wikipedia.org/wiki/Reverse_Polish_notation> for some background info.


Here comes the python code, but I doubt anyone is interested in it:-)


{% highlight python %}
import sys

class Calc:
    stack = []
    commands = {}
    def __init__(self):
        self.commands['p'] = (0, self.cmd_print)
        self.commands['q'] = (0, sys.exit)
        self.commands['+'] = (2, int.__add__)
        self.commands['-'] = (2, int.__sub__)
        self.commands['*'] = (2, int.__mul__)
        self.commands['/'] = (2, int.__div__)
        self.commands['%'] = (2, int.__mod__)
        self.commands['^'] = (2, int.__pow__)

    def run(self):
        while 1:
            self.parse(raw_input('> '))

    def parse(self, str):
        if str in self.commands:
            if self.commands[str][0] == 0:
                self.commands[str][1]()
            elif self.commands[str][0] == 2:
                if len(self.stack) < 2:
                    print "stack underflow"
                else:
                    self.cmd_binary(self.commands[str][1])
            else:
                try:
                    self.stack.append(int(str))
                except ValueError:
                    pass

    def cmd_print(self):
        for i in self.stack:
            print i

    def cmd_binary(self, method):
        b = self.stack.pop()
        a = self.stack.pop()
        self.stack.append(method(a,b))

if __name__ == '__main__':
    Calc().run()
{% endhighlight %}
