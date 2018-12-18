Title: Nodes and Associative Array fields
Date: 2018.12.19
Summary: A look at how associative array fields on nodes are handled and how they can affect performance
Category: Performance

Like a lot of other languages, BrightScript passes 'intrinsic' types by value and 'object' types by reference. In fact, [their documentation][ByRefDoc] shows an example of how this works:

<pre class="  language-vbnet"><code class="  language-vbnet"><span class="token keyword">function</span> Modify<span class="token punctuation">(</span>a <span class="token keyword">as</span> <span class="token keyword">Integer</span><span class="token punctuation">,</span> b <span class="token keyword">as</span> <span class="token keyword">Object</span><span class="token punctuation">)</span> <span class="token keyword">as</span> Void
    a <span class="token operator">=</span> <span class="token number">43</span>
    b.first <span class="token operator">=</span> <span class="token number">6</span>
<span class="token keyword">end</span> <span class="token keyword">function</span>
<span class="token comment">'.....</span>
x <span class="token operator">=</span> <span class="token number">42</span>
y <span class="token operator">=</span> { first<span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">,</span> second<span class="token punctuation">:</span> <span class="token number">2</span> }
Modify<span class="token punctuation">(</span>x<span class="token punctuation">,</span> y<span class="token punctuation">)</span>
<span class="token comment">' now x is still 42 but y.first is 6</span></code></pre>

As you can see, `y` was an associative array that got passed to the `Modify()` function. The `Modify()` function changed one of the properties on the `y` object, and since it was passed by ref, the original `y` object was updated.

But did you know that this is only true some of the time? Consider a simple component like this:

<pre class="  language-markup"><code class=" language-markup"><span class="token prolog">&lt;?xml version="1.0" encoding="utf-8" ?&gt;</span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>component</span> <span class="token attr-name">name</span><span class="token attr-value"><span class="token punctuation">=</span><span class="token punctuation">"</span>MyComponent<span class="token punctuation">"</span></span> <span class="token attr-name">extends</span><span class="token attr-value"><span class="token punctuation">=</span><span class="token punctuation">"</span>Group<span class="token punctuation">"</span></span><span class="token punctuation">&gt;</span></span>
    <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>script</span> <span class="token attr-name">type</span><span class="token attr-value"><span class="token punctuation">=</span><span class="token punctuation">"</span>text/brightscript<span class="token punctuation">"</span></span> <span class="token attr-name">uri</span><span class="token attr-value"><span class="token punctuation">=</span><span class="token punctuation">"</span>pkg:/components/MyComponent.brs<span class="token punctuation">"</span></span><span class="token punctuation">/&gt;</span></span>
    <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>interface</span><span class="token punctuation">&gt;</span></span>
        <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;</span>field</span> <span class="token attr-name">id</span><span class="token attr-value"><span class="token punctuation">=</span><span class="token punctuation">"</span>props<span class="token punctuation">"</span></span> <span class="token attr-name">type</span><span class="token attr-value"><span class="token punctuation">=</span><span class="token punctuation">"</span>assocarray<span class="token punctuation">"</span></span><span class="token punctuation">/&gt;</span></span>
    <span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>interface</span><span class="token punctuation">&gt;</span></span>
<span class="token tag"><span class="token tag"><span class="token punctuation">&lt;/</span>component</span><span class="token punctuation">&gt;</span></span></code></pre>

This component defines a single additional field called `props` that is of type `assocarray`. We use this component like this:

<pre class="  language-vbnet"><code class="  language-vbnet">    my <span class="token operator">=</span> m.top.findNode<span class="token punctuation">(</span><span class="token string">"my"</span><span class="token punctuation">)</span>
    my.props <span class="token operator">=</span> {
        text<span class="token punctuation">:</span> <span class="token string">"initial text"</span>
        count<span class="token punctuation">:</span> <span class="token number">1</span>
    }</code></pre>

And when we inspect the value, it is what we would expect:

<img src="/img/node_aa_1.png" />

Pretty straight-forward so far. And since we learned that AAs are passed by ref, we can modify the value like this:

`my.props.count = 2`

And when we inspect the value:

<img src="/img/node_aa_2.png" />

Wait, what?! We just set the value of the `count` property to 2, so why is it reporting it as 1?

The answer is that AAs are '[deep-copied through fields (pass-by-value)][DeepCopied]'. So in the special case where the AA is a field on a node, it is **not** passed by ref, but is instead copied. So when we did this:

`my.props.count = 2`

what really happened is that `my.props` returned a *copy* of the AA, on which we set the `count` property to 2. When we inspected `my.props` again, it was yet another copy which of course did *not* contain the changes.

So how do you actually update a property on an AA like this? You don't 😜 Instead, you have to update the entire AA. The common pattern is:

    props = my.props
    props.count = 2
    m.props = props

Here we get a copy of the AA, update it, and then set the node's field to the new AA. We can see that by doing this, the AA now reports the correct values:

<img src="/img/node_aa_3.png" />

There is one other caveat to this copying behavior. Consider a normal AA like this:

<pre class="  language-vbnet"><code class="  language-vbnet">    props <span class="token operator">=</span> {
        text<span class="token punctuation">:</span> <span class="token string">"initial text"</span>
        count<span class="token punctuation">:</span> <span class="token number">1</span>
        func<span class="token punctuation">:</span> <span class="token keyword">function</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
            <span class="token keyword">return</span> <span class="token string">"func called"</span>
        <span class="token keyword">end</span> <span class="token keyword">function</span>
    }</code></pre>

This AA defines `func` as a function, and when we call it, we get the expected results:

<img src="/img/node_aa_4.png" />

But if we set this AA as a node field and call it:

<pre class="  language-vbnet"><code class="  language-vbnet">    props <span class="token operator">=</span> {
        text<span class="token punctuation">:</span> <span class="token string">"initial text"</span>
        count<span class="token punctuation">:</span> <span class="token number">1</span>
        func<span class="token punctuation">:</span> <span class="token keyword">function</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
            <span class="token keyword">return</span> <span class="token string">"func called"</span>
        <span class="token keyword">end</span> <span class="token keyword">function</span>
    }
    my <span class="token operator">=</span> m.top.findNode<span class="token punctuation">(</span><span class="token string">"my"</span><span class="token punctuation">)</span>
    my.props <span class="token operator">=</span> props</code></pre>

<img src="/img/node_aa_5.png" />

The call fails, and when we inspect the AA, we see that the `func` property does not even exist! Every time the AA is copied, any function properties are silently stripped out (presumably to make them 'data-only' so that they can be marshalled across thread boundaries if necessary). Definitely something to watch out for since the debugger does not give any kind of hint that this has happened until you try to access the now-missing function.

As you might imagine, all of this has some substantial performance implications, especially when copying AAs across thread boundaries. Stay tuned for another post that dives deep into how to deal with those situations and some recommendations on best practices to ensure optimum performance.

[ByRefDoc]: https://sdkdocs.roku.com/display/sdkdoc/Component+Architecture#ComponentArchitecture-Intrinsictypesandobjecttypes
[DeepCopied]: https://sdkdocs.roku.com/display/sdkdoc/Optimization+Techniques#OptimizationTechniques-DataFlow
