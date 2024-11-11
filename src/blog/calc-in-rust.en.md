+++
title = "Simple calc in Rust"
date = 2023-02-09

[taxonomies]
tags = ["rust"]
+++

## Simple calculator programm in rust 

```rust
#[derive(Debug, Clone, PartialEq)]
enum Token<'a> {
    Number(&'a str),
    Sum(&'a str),
    Mul(&'a str),
    Curly(&'a str),
}

#[derive(Debug, Clone)]
enum Tree<'a> {
    Value(&'a str, bool),
    Group(Box<Tree<'a>>),
    Binary(Box<Tree<'a>>, &'a str, Box<Tree<'a>>),
}

fn lexer<'a>(mut input: &'a str) -> Result<Vec<Token<'a>>, String> {
    let mut tokens: Vec<Token> = Vec::new();

    while !input.is_empty() {
        if input.starts_with([' ', '\n', '\t']) {
            input = &input[1..];
            continue;
        }

        if let Some(idx) = input.find(|ch: char| !char::is_ascii_digit(&ch)) {
            if idx != 0 {
                tokens.push(Token::Number(&input[..idx]));
                input = &input[idx..];
                continue;
            }
        }

        if input.starts_with(['+', '-']) {
            tokens.push(Token::Sum(&input[..1]));
            input = &input[1..];
            continue;
        }

        if input.starts_with(['*', '/']) {
            tokens.push(Token::Mul(&input[..1]));
            input = &input[1..];
            continue;
        }

        if input.starts_with(['(', ')']) {
            tokens.push(Token::Curly(&input[..1]));
            input = &input[1..];
            continue;
        }

        return Err(format!("unexpected input `{}`", &input[..4]));
    }

    Ok(tokens)
}

fn main() {
    let input = "  -2 + 2 * -2 ";
    let tokens = lexer(input).unwrap();
    let tree = parse_expr(&tokens);

    println!("{:?}", tree);
    println!("{:?}", tree.as_ref().map(|(_, tree)|eval(tree)));
}

fn parse_expr<'a, 'b>(tok: &'b [Token<'a>]) -> Option<(&'b [Token<'a>], Tree<'a>)> {
    parse_factor(tok)
        .or_else(|| parse_sum(tok))
        .or_else(|| parse_atom(tok))
}

fn parse_factor<'a, 'b>(tok: &'b [Token<'a>]) -> Option<(&'b [Token<'a>], Tree<'a>)> {
    let (rest, atom) = parse_atom(tok)?;
    let (rest, tok) = match rest {
        [Token::Mul(sig), rest@..] => (rest, *sig),
        _ => return None,
    };
    let (rest, factor) = parse_atom(rest)?;

    Some((rest, Tree::Binary(Box::new(atom), tok, Box::new(factor))))
}

fn parse_sum<'a, 'b>(tok: &'b [Token<'a>]) -> Option<(&'b [Token<'a>], Tree<'a>)> {
    let (rest, atom) = parse_atom(tok)?;
    let (rest, tok) = match rest {
        [Token::Sum(sig), rest@..] => (rest, *sig),
        _ => return None,
    };
    let (rest, factor) = parse_factor(rest)?;

    Some((rest, Tree::Binary(Box::new(atom), tok, Box::new(factor))))
}

fn parse_atom<'a, 'b>(tok: &'b [Token<'a>]) -> Option<(&'b [Token<'a>], Tree<'a>)> {
    match tok {
        [Token::Sum("-"), Token::Number(val), rest@..] => Some((rest, Tree::Value(*val, true))),
        [Token::Number(val), rest@..] => Some((rest, Tree::Value(*val, false))),
        [Token::Curly("("), rest @ ..] => {
            let (rest, tree) = parse_expr(rest)?;

            if rest.get(0)? == &Token::Curly(")") {
                return Some((&rest[1..], Tree::Group(Box::new(tree))));
            }

            None
        },

        _ => None
    }
}

fn eval<'a>(tree: &Tree<'a>) -> Option<i32> {
    match tree {
        Tree::Value(val, is_neg) => {
            let val: i32 = val.parse().unwrap();
            let val = if *is_neg {-val} else {val};
            Some(val)
        },
        Tree::Group(inner) => eval(inner),
        Tree::Binary(left, op, right) => {
            let left = eval(left)?;
            let right = eval(right)?;

            match *op {
                "*" => Some(left * right),
                "/" => Some(left / right),
                "+" => Some(left + right),
                "-" => Some(left - right),
                _ => None
            }
        }
    }
}
```
