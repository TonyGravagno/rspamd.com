---
layout: doc
title: Rspamd Composites
---

# Rspamd composite symbols
{:.no_toc}

Rspamd composites are ([symbols](metrics.html#symbols) used to combine rules and create more complex rules. 

<div id="toc" markdown="1">
  <h2 class="toc-header">Contents</h2>
  * TOC
  {:toc}
</div>


## Configuration

Composite rules are defined in the `composites` section of the configuration. Use `local.d/composites.conf` (which effects changes **inside** the `composites` section) to define new composites or to change/override existing composites.

All symbols, including composites, are objects. The object key is the name of the composite. Any name is acceptable, by convention using upper-case letters, numbers, or underscores. Actual examples include: MAIL_RU_MAILER_BASE64 and RCVD_DKIM_ARC_DNSWL_MED. The object value defines the composite's properties.

## Composite expressions

The composite includes an "expression" property, which is a pattern that makes use of other rules ([symbols](metrics.html#symbols) or even other composites). Think of it like an IF/THEN structure in any programming language. When the expression evaluates to true, the composite triggers (also called "matches"), just like any other rule/symbol. The "score" property and other details add an additional +/- weight for the composite to the overall message weight. For example, you can define a composite that fires when two specific symbols are found, and **replace** these symbols and their weights with the "score" value of the composite:

~~~ucl
TEST_COMPOSITE {
    expression = "SYMBOL1 and SYMBOL2";
    score = 5.0;
}
~~~

In this case, if a message has both `SYMBOL1` and `SYMBOL2`, they are replaced by symbol `TEST_COMPOSITE`. The weights of `SYMBOL1` and `SYMBOL2` are subtracted from the overall metric/weight accordingly.

Note the terms 'weight' and 'score' are synonymous for an individual rule, though more generally, a score is associated with a single rule or composite, and a message has an overall weight or metric from all scores.

You can use the following operations in a composite expression:

* `AND` `&` - matches true only if both operands are true
* `OR` `|` - matches true if any operands are true
* `NOT` `!` - matches true if operand is false

You also can use round brackets (parentheses) to define priorities. Otherwise operators are evaluated from left to right. For example:

~~~ucl
COMP1 {
    expression = "SYMBOL1 and SYMBOL2 and ( not SYMBOL3 | not SYMBOL4 | not SYMBOL5 )";
    score = 10.0;
    group = "Some group";
}
~~~

A composite rule can include other composites (symbols) in the expression. There is no restriction on definition order:

~~~ucl
COMP2 {
    expression = "SYMBOL1 AND COMP3";
}
COMP3 {
    expression = "SYMBOL2 OR NOT SYMBOL3";
}
~~~

Composites should not be recursive, but this is normally detected and avoided by Rspamd automatically. In the above example, if COMP3 tested for COMP2 then the two composites could potentially process recursively.

When a composite is triggered, by default Rspamd **removes** the symbols and weights used in the composite expression. The idea is to avoid double-counting scores of both the symbols that are matched **and** the composite(s) that they trigger in turn. However, composite policies can override the defaults:

To modify the removal policy for a single symbol, prefix symbol names in the expression with one of these special characters:
    * `~`: remove symbol only, weight is preserved : ~SYMBOL1
    * `-`: do not remove anything, weight and symbol itself are preserved : -SYMBOL2
    * `^`: force removal of symbol and weight : ^SYMBOL3
      - By default, Rspamd prefers to leave symbols when one composite wants to remove and another composite wants to leave any name or score. This option overrides that. This is discussed further, below.

## One removal policy for all symbols

To modify the default policy for **all** symbols in the expression of a composite, use the `policy` property:
    * `default`: default policy - remove weight and symbol (not required to be explicit)
    * `leave`: leave both symbol and score
    * `remove_symbol`: remove symbol only
    * `remove_weight`: remove score only

Examples (and more follow in the next section) :

~~~ucl
COMP1 {
    expression = "SYMBOL1 and SYMBOL2";
    policy = "leave";
# Leave both symbols and their scores - the composite is also reported.
}
COMP2 {
    expression = "SYMBOL3 and SYMBOL4";
    policy = "remove_weight";
# Symbols are still removed, combined weight remains and the composite is reported.
}
COMP3 {
    expression = "~SYMBOL5 & !SYMBOL6";
# SYMBOL5 must be triggered. SYMBOL5 will be removed, but its score will not be.
# SYMBOL6 must not have been triggered (there will be no symbol or score for SYMBOL6).
# So COMP3 will be logged with the score of SYMBOL5.
}
COMP4 {
    expression = "SYMBOL5 &! SYMBOL6";
    score = 1.0;
# Similar to COMP3, but the score of SYMBOL5 is not used, the new score is used.
# Recall that by default the symbol and score are removed unless explicitly preserved.
# If there is no score, COMP4 would still be reported, but with no score.
}
~~~

Symbols are removed **after** all composite processing. Hence, in one composite you cannot assume that some symbol has been removed by another composite.

## Composite weight rules

Composites can log symbols as a part of the final metric, and/or record their weights. Here are more examples of how to control the inclusion of symbols and their scores. For these examples, consider symbols `A` and `B` are both matched, with weights `W_a` and `W_b`, and a composite `C` is thus also matched with weight `W_c` set in its `score` property.

* If the `C` expression is `A & B`, then both symbols are *removed*, and their weights are removed as well. This results in a final metric with a single symbol `C` with weight `W_c`.
* If the `C` expression is `-A & B`, then rule `A` is preserved in the final metric, and the symbol `C` is reported. The weight of `A` is preserved as well, so the total weight of `-A & B` will be `W_a + W_c` (weight of `B` is still removed).
* If the `C` expression is `~A & B`, then rule `A` is removed, but it's weight is preserved.The final metric only shows `C`, with a total weight of `W_a + W_c`

When you have multiple composites which include the same symbol and one composite wants to remove the symbol and another composite wants to preserve it, the symbol is preserved by default. Here are some more examples:

~~~ucl
COMP1 {
    expression = "BLAH | !DATE_IN_PAST";
}
COMP2 {
    expression = "!BLAH | -DATE_IN_PAST";
}
COMP3 {
    expression = "!BLAH | DATE_IN_PAST";
}
~~~

Both `BLAH` and `DATE_IN_PAST` exist in the message's check results. By default the symbols would be removed whenever these composites are triggered. However, `COMP2` wants to preserve `DATE_IN_PAST` so it will be saved in the output. As noted above, the order in which composites are defined is not important - even though `COMP3` directs the symbol to be removed when the composite matches, the symbol will remain because of the `COMP2` override.

If we rewrite the previous example but replace `-` with `~` then `DATE_IN_PAST` will be removed (however, its weight won't be removed):

~~~ucl
COMP1 {
    expression = "BLAH | !DATE_IN_PAST";
}
COMP2 {
    expression = "!BLAH | ~DATE_IN_PAST";
}
COMP3 {
    expression = "!BLAH | DATE_IN_PAST";
}
~~~

When we want to remove a symbol, despite other composite directives, add the prefix `^` to the symbol:

~~~ucl
COMP1 {
    expression = "BLAH | !DATE_IN_PAST";
}
COMP2 {
    expression = "!BLAH | ^DATE_IN_PAST";
}
COMP3 {
    expression = "!BLAH | -DATE_IN_PAST";
}
~~~

In this example `COMP3` wants to save `DATE_IN_PAST`, however `COMP2` overrides this with the "**^** force removal" directive, and removes `DATE_IN_PAST`. As noted above, the order in which composites are defined is not important.

## Composites with symbol groups

A group of symbols can be used in a composite rule. This effectively means **any** matched symbol of the specified group:

* `g:<group>` - matches **any** symbol
* `g+:<group>` - matches any symbol with a **positive** score
* `g-:<group>` - matches any symbol with a **negative** score

Removal policies are applied only to the matched symbols and not to the entire group.

In the following example, `COMP1` is triggered when the single `SYMBOL2` is matched, and when there are no symbols matched in the `mua` group, and when any symbol in the `fuzzy` group matches with a positive score.

~~~ucl
COMP1 {
    expression = "SYMBOL2 & !g:mua & g+:fuzzy";
}
~~~

Here is another example:  
The following is an actual composite, where only policy symbols with a negative score are matched (message follows good policy rules), and yet there are positive (bad) matches in any of the other groups. When this is the case, the symbols representing the good policies are removed, but their weight is preserved (note the "~" directive), and where the other symbols would normally be removed, the "-" directs them to be preserved. This composite is a coded way of saying "the message seems to be good, and we'll give it credit for that, but there are some other issues with it that should also negatively affect its score." There is also a score of 0.1 declared for this composite which is more fine-tuning to say the message is being slightly weighted as a penalty for this awkward condition.

~~~ucl
BAD_REP_POLICIES {
  description = "Contains valid policies but are also marked by fuzzy/bayes/surbl/rbl";
  expression = "(~g-:policies) & (-g+:fuzzy | -g+:bayes | -g+:surbl | -g+:rbl)";
  score = 0.1;
}
~~~

Try not to get confused with the special characters:
* The ~/- before the "g" refer to the removal policy of related symbols and/or weights.
* The +/- after the "g" refer to bad/good scores of symbols within the groups.

## Disabling composites

You can disable a composite with the property `enabled = false;`. For example, to disable the `DKIM_MIXED` composite defined in the stock configuration, you could add the following to `local.d/composites.conf`:

~~~ucl
DKIM_MIXED {
    enabled = false;
}
~~~

Note that you do not need to set the expression in your local.d file. The local.d files specify overrides of details, not complete replacements. The `enabled` property overrides the default value of "true". Since this override does not include any other details, the details remain unchanged from their defaults in composites.conf.

As of Rspamd v1.9, composites can also be disabled in [users settings](settings.html).

## Composites on symbol options

As of Rspamd v2.0, expressions can be augmented with required symbol options. For example, if a symbol `SYM` can insert options `opt1` and `opt2` one can create a composite expression that works merely if `opt2` option is presented:

~~~ucl
TEST2 {
    expression = "SYM[opt2]";
}
~~~

`[opt2]` syntax means a list of options allowed for a symbol to match. You can also add multiple options:

`[opt1,opt2]` - this means **both** `opt1` and `opt2` must be added by a symbol,

or even regular expressions:

`[/opt\d/i]` - this must not include a comma, even escaped...

or a mix of both:

`[/opt\d/i, foo]`

In all cases, where a list of options is specified, **all** matches are required.

In the future, this could be extended to support fully functional expressions.
