---


---

<h1 id="extracting-variables-from-email-templates">Extracting Variables from Email Templates</h1>
<blockquote>
<p><em><strong>tl; dr; -</strong> Use <code>{{[\s]*[#/]?[a-zA-Z][a-zA-Z0-9\.]*[\s]*[a-zA-Z0-9\.]*[\s]*}}</code> to<br>
identify {{Handlebars}} style variables. Works with ES6’s regex engine. Caveats apply. See below.</em></p>
</blockquote>
<p>One of the big challenges encountered while testing SES email templates is knowing what template data to use in order to correctly send test emails.</p>
<p>Famously, SES will refuse to deliver emails to customers if ALL the necessary template data is not supplied to it. It does not gracefully replace unavailable data with an empty string.</p>
<p>Since most email templates use {{<a href="https://handlebarsjs.com/">Handlebars</a>}} or some other <a href="http://mustache.github.io/">Mustache</a> based template engine, the below post will be about extracting {{Handlebars}} variables from a template.</p>
<p>To me, it seems like SES does not internally use Handlebarsjs to compile and use templates because if they did, they would successfully compile templates even if some variables are missing. Here is why.</p>
<p>Say, for instance, that your email template looks like this:</p>
<pre class=" language-html"><code class="prism  language-html"><span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>p</span><span class="token punctuation">&gt;</span></span>Hello {{ name }}, happy {{ day_of_week }}<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>p</span><span class="token punctuation">&gt;</span></span>
</code></pre>
<p>If you tried to send an email through SES but neglected to include the <code>{{name}}</code> key-value pair in your template data, SES would simply not send the email to the recipient.</p>
<p>If you tried to compile the above template using Handlebarsjs while missing <code>{{name}}</code> but including <code>day_of_week</code> as FRIDAY, you would get</p>
<pre class=" language-html"><code class="prism  language-html"><span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>p</span><span class="token punctuation">&gt;</span></span>Hello , happy FRIDAY<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>p</span><span class="token punctuation">&gt;</span></span>
</code></pre>
<p>Once the template has been compiled, you could argue either way - that there is no reason not to send the email out if it is missing some data OR that getting the email out, warts and all is fine.</p>
<p>In any case SES does not send out the email, it suggests to me that SES does not use Handlebarsjs internally.</p>
<p>This brings us to the next issue.</p>
<h3 id="identifying-variables">Identifying {{variables}}</h3>
<p>How do we make sure we supply SES with all the necessary variables so that it can send out the email?</p>
<p>With some trial an error, I came up with <code>{{[\s]*[#/]?[a-zA-Z][a-zA-Z0-9\.]*[\s]*[a-zA-Z0-9\.]*[\s]*}}</code>. Let’s look at what type of strings this regex is able to find.</p>
<h4 id="correct">Correct:</h4>
<p>The regex treats the following as acceptable matches.</p>
<pre><code>1. {{data}}
2. {{ data2.point }}
3. {{ #if }}
4. {{#each this.data.point }}
5. {{/each }}
</code></pre>
<h4 id="incorrect">Incorrect:</h4>
<p>The regex treats the following as acceptable matches but that is incorrect from Handlebars’ point of view.</p>
<pre><code>1. {{data.}} ==&gt; trailing period in not valid Handlebars syntax
2. {{ #data2.poi..nt}} ==&gt; multiple adjacent periods within the same string
3. {{ #i.f }} ==&gt; does not check for reserved keywords like `if`, `else` and `each`
</code></pre>
<h1 id="the-upshot">The Upshot</h1>
<p>Since the regular expression sequentially identifies all <code>{{strings}}</code>, it becomes MUCH easier to form your template object.</p>
<p>For example, in the following <code>{{strings}}</code> array, note the strings marked with asterisks around them like <code>*{{#each item}}*</code>. This immediately tells the person testing an SES template that they can use an array of items with the JSON key <code>item</code> to add more than one item to the email.</p>
<p>We feel this is of much more utility than simply showing users required variables.</p>
<pre><code>{{CustomerName}}
{{OrderNumber}}
{{EstimatedDeliveryDate}}
*{{#each item}}*
{{this.ProductImage}}
{{this.ProductName}}
*{{#if DiscountIsApplied }}*
{{this.ProductPrice}}
*{{/if}}*
{{this.ProductPriceAfterDiscount}}
{{this.Description}}
*{{#if this.Subscription}}*
{{this.Subscription.SubscriptionDescription}}
*{{else}}*
{{this.PackQuantity}}
*{{/if}}*
*{{/each}}*
{{OrderNumber}}
{{OrderDate}}
{{OrderSubTotalInclTax}}
{{CodCharge}}
{{TotalDiscount}}
{{OrderGrandTotal}}
</code></pre>
<h2 id="conclusion">Conclusion</h2>
<p>The regular expression on the page is not useful for error checking. It is simply a tool to quickly identify and extract <code>{{strings}}</code> which look like Handlebars variables from a template so it should be treated as such.</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

