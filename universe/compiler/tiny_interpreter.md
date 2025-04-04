---
title: 简易解释器
author: 66RING
date: 2025-04-03
tags: 
- compiler
mathjax: true
---


# 简易解释器

> [Let’s Build A Simple Interpreter.](https://ruslanspivak.com/lsbasi-part1/)


- Terms
    * 词法分析, Tokenize, lexer
        + 字符串匹配, 映射成约定好的原语(string -> [Token]) 
            + `{type: xxx, value: xxx, ....}`
    * 语法分析, parser
- 词法分析
    * 正则表达式
    * [type: reg, ...], 便利匹配即可
- 语法分析
    * EBNF

```
program -> stmt* # 程序由多条语句组成
stmt -> expr     # 每个语句都是一个表达式
expr -> bin_expr # 表达式是二元表达式
bin_expr -> add_expr # 二元表达式是加法表达式
add_expr -> mul_expr((PLUS|MIN mul_expr)*) # 加法表达式可以是: NUM(mul_expr)跟上多个+-NUM(mul_expr)
mul_expr -> NUM((MUL|DIV)NUM)* # 乘法表达式可以是: NUM跟上多个*/NUM, 因为乘除的优先级高于加减, 所以更接近叶子
NUM -> FACTOR  # NUM可以是整数或者浮点数，更抽象的表示
FACTOR -> INT  # 整数
```

根据语法规则逐级抽象和构建AST(parse)即可


## lexer

```python
from enum import Enum, auto
import re

class TokenType(Enum):
    NUMBER = auto()
    OPERATOR = auto()
    OPEN_PAREN = auto()
    CLOSE_PAREN = auto()
    VARIABLE = auto()
    BLANK = auto()
    STRING = auto()

    EOF = auto()  # End of File

class Token:
    def __init__(self, type: TokenType, value: str):
        self.type = type
        self.value = value

    def __repr__(self):
        return f"Token({self.type}, {self.value})"

TOKEN_REGEX = {
    TokenType.NUMBER: r"\d+(\.\d+)?",
    TokenType.OPERATOR: r"[+\-*/]",
    TokenType.OPEN_PAREN: r"\(",
    TokenType.CLOSE_PAREN: r"\)",
    TokenType.VARIABLE: r"[a-zA-Z_]+[a-zA-Z0-9_]*",
    TokenType.BLANK: r"[ \t\n\r]",
    TokenType.STRING: r"\".*?\"",
}

class Lexer:
    def tokenize(self, expression) -> list[Token]:
        tokens = []
        while expression:
            match = None
            # Try to match each token type
            for token_type, pattern in TOKEN_REGEX.items():
                match = re.match(pattern, expression)

                if match:
                    value = match.group(0)
                    expression = expression[len(value):]

                    if token_type == TokenType.STRING:
                        tokens.append(Token(token_type, value[1:-1]))
                    elif token_type == TokenType.BLANK:
                        # Ignore blank spaces
                        pass
                    else:
                        tokens.append(Token(token_type, value))

                    break


            if not match:
                raise Exception(__file__, f"Unexpected character: {expression[0]}")

        tokens.append(Token(TokenType.EOF, ""))
        return tokens


def main():
    expr = input(">>>")
    lexer = Lexer()
    tokens = lexer.tokenize(expr)
    print(tokens)

if __name__ == "__main__":
    main()
```

## ast

```python
from enum import Enum, auto

class NodeType(Enum):
    # AST nodes
    PROGRAM = auto()
    STMT = auto()

    EXPR = auto()
    BIN_EXPR = auto()
    ADD_EXPR = auto()
    MUL_EXPR = auto()

    NUM = auto()
    FACTOR = auto()
    INT = auto()

class Program:
    def __init__(self):
        self.type: NodeType = NodeType.PROGRAM
        self.body: list[Stmt] = []

    def __repr__(self) -> str:
        res = "{\n"
        res += f"\t{self.type.name}\n"
        for stmt in self.body:
            res += f"{stmt.__repr__(1)}\n"
        res += "}\n"
        return res


class Stmt:
    def __init__(self):
        self.type: NodeType = NodeType.STMT

    def __repr__(self, indent) -> str:
        res = "\t" * indent + "{\n"
        res += "\t" * (indent+1) + f"{self.type.name}\n"
        # NOTE: no sub tree
        res += "\t" * indent + "}\n"
        return res


class Expr(Stmt):
    def __init__(self):
        self.type: NodeType = NodeType.EXPR

class BinExpr(Expr):
    def __init__(self, left: Expr, operator: str, right: Expr):
        self.type: NodeType = NodeType.BIN_EXPR
        self.left: Expr = left
        self.operator: str = operator
        self.right: Expr = right

    def __repr__(self, indent) -> str:
        res = "\t" * indent + "{\n"
        res += "\t" * (indent+1) + f"{self.type.name}\n"
        # NOTE: indent sub tree
        res += f"{self.left.__repr__(indent+1)}"
        res += "\t" * (indent+1) + f"{self.operator}\n"
        res += f"{self.right.__repr__(indent+1)}"
        res += "\t" * indent + "}\n"
        return res


class Number(Expr):
    def __init__(self, value: str):
        self.type: NodeType = NodeType.NUM
        self.value: str = value

    def __repr__(self, indent) -> str:
        res = "\t" * indent + "{\n"
        res += "\t" * (indent+1) + f"{self.type.name}\n"
        res += "\t" * (indent+1) + f"{self.value}\n"
        # NOTE: no sub tree
        res += "\t" * indent + "}\n"
        return res
```

## parser

```python
from lexer import Token, TokenType, Lexer
from ast import Number, Program, Stmt, Expr, BinExpr

'''
parser

program -> stmt* # a program consists of multiple statements
stmt -> expr     # each statement is an expression
expr -> bin_expr # each expression is a binary expression
bin_expr -> add_expr # binary expression is an addition expression
add_expr -> mul_expr((PLUS|MIN mul_expr)*) # addition expression can be: mul_expr followed by multiple +/mul_expr, because addition and subtraction have the same priority, so they are closer to the leaf
mul_expr -> NUM((MUL|DIV)NUM*) # multiplication expression can be: NUM followed by multiple *NUM, because multiplication and division have the same priority, so they are closer to the leaf
NUM -> INT  # NUM is a factor which can be a number, variable, or parenthesized expression
'''


class Parser:
    def __init__(self) -> None:
        pass

    def tk(self) -> Token:
        return self.tokens[self.pos]

    def eat(self) -> Token:
        token = self.tk()
        self.pos += 1
        return token

    def expect(self, token_type: TokenType) -> Token:
        if self.tk().type == token_type:
            return self.eat()
        else:
            raise Exception(__file__, f"Expected {token_type}, but got {self.tk().type}")

    def parse(self, expr: str) -> None:
        lexer = Lexer()
        self.tokens = lexer.tokenize(expr)
        self.pos = 0

        return self.parse_program()

    def parse_program(self) -> Program:
        '''
            program -> stmt*
        '''
        program = Program()
        while self.tk().type != TokenType.EOF:
            program.body.append(self.parse_stmt())
        return program

    def parse_stmt(self) -> Stmt:
        '''
            stmt -> expr
        '''
        return self.parse_expr()

    def parse_expr(self) -> Expr:
        '''
            expr -> bin_expr
        '''
        return self.parse_bin_expr()

    def parse_bin_expr(self) -> Expr:
        '''
            bin_expr -> add_expr
        '''
        return self.parse_add_expr()


    def parse_add_expr(self) -> Expr:
        '''
            add_expr -> mul_expr((PLUS|MIN mul_expr)*)
        '''
        left = self.parse_mul_expr()

        while self.tk().value in ('+', '-'):
            op = self.eat()
            right = self.parse_mul_expr()
            left = BinExpr(left, op, right)
        return left

    def parse_mul_expr(self) -> Expr:
        '''
            mul_expr -> NUM((MUL|DIV)NUM*)
        '''
        left = self.parse_num()
        while self.tk().value in ('*', '/'):
            op = self.eat()
            right = self.parse_num()
            left = BinExpr(left, op, right)
        return left

    def parse_num(self) -> Expr:
        '''
            NUM -> INT
        '''
        if self.tk().type == TokenType.NUMBER:
            return Number(self.eat().value)
        elif self.tk().type == TokenType.OPEN_PAREN:
            self.eat()
            expr = self.parse_expr()
            self.expect(TokenType.CLOSE_PAREN)
            return expr

        raise Exception(__file__, f"Expected NUM, but got {self.tk().type}")



def main():
    while True:
        expr = input(">>>")
        if expr == ".q":
            break
        parser = Parser()
        program = parser.parse(expr)
        print(program)

if __name__ == "__main__":
    main()
```







