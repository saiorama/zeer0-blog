# Extracting Variables from Email Templates

> ***tl; dr; -** Use `{ {[\s]*[#/]?[a-zA-Z][a-zA-Z0-9\.]*[\s]*[a-zA-Z0-9\.]*[\s]*}}` to
> identify { {Handlebars}} style variables. Works with ES6's regex engine. Caveats apply. See below.*

*(remove the blank space between { { before using the regex)*

One of the big challenges encountered while testing SES email templates is knowing what template data to use in order to correctly send test emails.

Famously, SES will refuse to deliver emails to customers if ALL the necessary template data is not supplied to it. It does not gracefully replace unavailable data with an empty string.

Since most email templates use { {[Handlebars](https://handlebarsjs.com/)}} or some other [Mustache](http://mustache.github.io/) based template engine, the below post will be about extracting { {Handlebars}} variables from a template.

To me, it seems like SES does not internally use Handlebarsjs to compile and use templates because if they did, they would successfully compile templates even if some variables are missing. Here is why.

Say, for instance, that your email template looks like this:

```html
<p>Hello { { name }}, happy { { day_of_week }}</p>
```
If you tried to send an email through SES but neglected to include the `{ {name}}` key-value pair in your template data, SES would simply not send the email to the recipient.

If you tried to compile the above template using Handlebarsjs while missing `{ {name}}` but including `day_of_week` as FRIDAY, you would get 
```html 
<p>Hello , happy FRIDAY</p>
```

Once the template has been compiled, you could argue either way - that there is no reason not to send the email out if it is missing some data OR that getting the email out, warts and all is fine. 

In any case SES does not send out the email, it suggests to me that SES does not use Handlebarsjs internally.

This brings us to the next issue.
### Identifying {{variables}}
How do we make sure we supply SES with all the necessary variables so that it can send out the email?

With some trial an error, I came up with `{ {[\s]*[#/]?[a-zA-Z][a-zA-Z0-9\.]*[\s]*[a-zA-Z0-9\.]*[\s]*}}`. Let's look at what type of strings this regex is able to find.

#### Correct:
The regex treats the following as acceptable matches.
```
1. { {data}}
2. { { data2.point }}
3. { { #if }}
4. { {#each this.data.point }}
5. { {/each }}
```
#### Incorrect: 
The regex treats the following as acceptable matches but that is incorrect from Handlebars' point of view.
```
1. { {data.}} ==> trailing period in not valid Handlebars syntax
2. { { #data2.poi..nt}} ==> multiple adjacent periods within the same string
3. { { #i.f }} ==> does not check for reserved keywords like `if`, `else` and `each`
```

# The Upshot
Since the regular expression sequentially identifies all `{ {strings}}`, it becomes MUCH easier to form your template object. 

For example, in the following `{ {strings}}` array, note the strings marked with asterisks around them like `*{ {#each item}}*`. This immediately tells the person testing an SES template that they can use an array of items with the JSON key `item` to add more than one item to the email.

We feel this is of much more utility than simply showing users required variables.

```
{ {CustomerName}}
{ {OrderNumber}}
{ {EstimatedDeliveryDate}}
*{ {#each item}}*
{ {this.ProductImage}}
{ {this.ProductName}}
*{ {#if DiscountIsApplied }}*
{ {this.ProductPrice}}
*{ {/if}}*
{ {this.ProductPriceAfterDiscount}}
{ {this.Description}}
*{ {#if this.Subscription}}*
{ {this.Subscription.SubscriptionDescription}}
*{ {else}}*
{ {this.PackQuantity}}
*{ {/if}}*
*{ {/each}}*
{ {OrderNumber}}
{ {OrderDate}}
{ {OrderSubTotalInclTax}}
{ {CodCharge}}
{ {TotalDiscount}}
{ {OrderGrandTotal}}
```


## Conclusion
The regular expression on the page is not useful for error checking. It is simply a tool to quickly identify and extract `{ {strings}}` which look like Handlebars variables from a template so it should be treated as such.

> Written with [StackEdit](https://stackedit.io/).
