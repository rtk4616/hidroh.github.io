---
layout: post
title:  "Rich text battle: TextView vs WebView"
date:   2016-02-27 14:53:34
description: "An analysis of using TextView versus WebView to display HTML content"
image: /assets/img/rendering_webview.png
tags: android view textview webview performance html memory gpu cpu
github: richtext
---

<div class="cap"></div>

So, say we have a piece of rich text, most probably in the form of HTML (e.g. a simplified HTML made for mobile reading), what kind of widget is best used to display it? How do we achieve the flexible yet rich reading experience from popular mobile readers like Readability, Pocket, or [Materialistic] (!)?

In this article, we will explore two very popular and powerful widgets that has been available since API 1: [`TextView`][TextView] and [`WebView`][WebView], for the purpose of rich text rendering. The article does not aim to provide a comprehensive comparison, but rather touches on several critical desicion making points.

<!--more-->[ ](#){: id="more"}

For the sake of comparison, let's assume that we are given the task of displaying [the following image-intensive article][example], styled to specific background color, text size and color. We will first try to use available APIs in `TextView` and `WebView` to render the given HTML in the same, comparable way, then analyze their performance: memory, GPU and CPU consumption. Example code can be found [here][Github].

<a href="/assets/img/textview-webview-sample.png"><img src="/assets/img/textview-webview-sample.png" class="img-responsive center-block img-thumbnail" style="max-height:320px" /></a>

### The basics

First let's quickly get through the basics. By the way they are named, it is quite obvious that `TextView` is meant for text rendering, while `WebView` is for webpage rendering. With proper use of their APIs however, we can quickly turn both of them into flexible widgets that can render rich text with images.

* Use [`Html.fromHtml(String, ImageGetter, TagHandler)`][fromHtml] to parse HTML string into `TextView`
* Use [`WebView.loadDataWithBaseURL()`][loadDataWithBaseURL] to load HTML string into `WebView`

**TextView**

`TextVew`, with default support for [`Spanned`][Spanned], an interface for markup text, allows very fine-grained format options over any range of text. It also exposes a bunch of styling attributes, e.g. `textAppearance`, and APIs for controlling text appearance. Check out Flavien Laurent[^flaurent]'s and Chiu-Ki Chan[^chiuki]'s excellent materials on advanced uses of `TextView` and `Spanned`.

When it comes to rich text however, `TextView` shows certain limitations that we may want to weigh up before considering using it: it only handles a [limited set of HTML tags][TextView tags], which should be sufficient in most cases; and we have to handle fetching embedded remote images by ourselves, via an [`ImageGetter`][ImageGetter] instance; and intercept hyperlinks by using [`LinkMovementMethod`][LinkMovementMethod]. Whew!

<a href="#codeTextView" class="btn btn-default" data-toggle="collapse">Toggle code <i class="fa fa-code"></i></a>

<div class="collapse" id="codeTextView">
{% highlight java %}
textView.setMovementMethod(new LinkMovementMethod());
textView.setText(Html.fromHtml("<p>Hello World!</p>",
        new PicassoImageGetter(textView), null));

class PicassoImageGetter implements Html.ImageGetter {
    private final TextView mTextView;
    ...
    @Override
    public Drawable getDrawable(String source) {
        URLDrawable d = new URLDrawable(mTextView.getResources(), null);
        new LoadFromUriAsyncTask(mTextView, d)
                .execute(Uri.parse(source));
        return d;
    }
}
class LoadFromUriAsyncTask extends AsyncTask<Uri, Void, Bitmap> {
    private final WeakReference<TextView> mTextViewRef;
    private final URLDrawable mUrlDrawable;
    ...
    @Override
    protected Bitmap doInBackground(Uri... params) {
        try {
            return Picasso.with(mTextViewRef.get().getContext())
                    .load(params[0]).get();
        } catch (IOException e) {
            return null;
        }
    }

    @Override
    protected void onPostExecute(Bitmap result) {
        ...
        TextView textView = mTextViewRef.get();
        mUrlDrawable.mDrawable = new BitmapDrawable(
                textView.getResources(), result);
        ...
        textView.setText(textView.getText());
    }
}
class URLDrawable extends BitmapDrawable {
    private Drawable mDrawable;
    ...
    @Override
    public void draw(Canvas canvas) {
        if(mDrawable != null) {
            mDrawable.draw(canvas);
        }
    }
}
{% endhighlight %}
</div>

**WebView**

Meant for HTML display, `WebView` supports most HTML tags out of the box. We already know how to use a `WebView` to load a remote webpage with `WebView.loadUrl()`, but it can also load a local webpage as well: by wrapping HTML string inside a `<body>` block and loading it via `WebView.loadDataWithBaseURL()`, where base URL is `null`. `WebView` supports zooming (need to enable), and handles images and hyperlinks by default (of course!).


<a href="#codeWebViewSimple" class="btn btn-default" data-toggle="collapse">Toggle code <i class="fa fa-code"></i></a>

<div class="collapse" id="codeWebViewSimple">
{% highlight java %}
webView.getSettings().setBuiltInZoomControls(true); // optional
webView.loadDataWithBaseURL(null,
        wrapHtml(webView.getContext(), "<p>Hello World!</p>"),
        "text/html", "UTF-8", null);

private String wrapHtml(Context context, String html) {
    return context.getString(R.string.html, html);
}
{% endhighlight %}

{% highlight xml %}
<resources>
    <string name="html" translatable="false">
        <![CDATA[
        <html>
            <head></head>
            <body>%1$s</body>
        </html>"
        ]]>
    </string>
</resources>
{% endhighlight %}
</div>

### Styling

Now let's try to style `TextView` and `WebView` to render the example article on a teal theme, with some standard paddings around it. We can see that both of them are capable of rendering the page in pretty much the same way, with the exception of `TextView` ignoring the `<hr />` tag it cannot handle.

<div class="container">
    <div class="row">
        <div class="col-xs-10 col-xs-offset-1 col-md-8 col-md-offset-1">
            <img src="/assets/img/rendering_textview.png" style="max-width:45%" />
            <img src="/assets/img/rendering_webview.png" style="max-width:45%" />
        </div>
    </div>
</div>

<figcaption>Styling: TextView (left) vs WebView (right)</figcaption>

While `TextView` provides many attributes and APIs out of the box for styling, `WebView` does not provide public APIs for styling its HTML content. However with some basic CSS knowledge, one can instrument given HTML with CSS styles and achieve desired styling as above. We need to be careful on the conversion from CSS metrics to Android metrics though.

<a href="#codeWebViewFull" class="btn btn-default" data-toggle="collapse">Toggle code <i class="fa fa-code"></i></a>

<div class="collapse" id="codeWebViewFull">

{% highlight java %}
webView.loadDataWithBaseURL(null,
                wrapHtml(webView.getContext(),
                        "<p>Hello World!</p>",
                        android.R.attr.textColorTertiary,
                        android.R.attr.textColorLink,
                        getResources().getDimension(R.dimen.text_size),
                        getResources().getDimension(R.dimen.activity_horizontal_margin)),
                "text/html", "UTF-8", null);

private String wrapHtml(Context context, String html,
                        @AttrRes int textColor,
                        @AttrRes int linkColor,
                        float textSize,
                        float margin) {
    return context.getString(R.string.html,
            html,
            toHtmlColor(context, textColor),
            toHtmlColor(context, linkColor),
            toHtmlPx(context, textSize),
            toHtmlPx(context, margin));
}

private String toHtmlColor(Context context, @AttrRes int colorAttr) {
    return String.format("%06X", 0xFFFFFF &
            ContextCompat.getColor(context, getIdRes(context, colorAttr)));
}

private float toHtmlPx(Context context, float dimen) {
    return dimen / context.getResources().getDisplayMetrics().density;
}

@IdRes
private int getIdRes(Context context, @AttrRes int attrRes) {
    TypedArray ta = context.getTheme().obtainStyledAttributes(new int[]{attrRes});
    int resId = ta.getResourceId(0, 0);
    ta.recycle();
    return resId;
}
{% endhighlight %}
{% highlight xml %}
<resources>
    <string name="html" translatable="false">
        <![CDATA[
        <html>
            <head>
                <style type="text/css">
                    body {
                        font-size: %4$fpx;
                        color: #%2$s;
                        margin: %5$fpx %5$fpx %5$fpx %5$fpx;
                    }
                    a {color: #%3$s;}
                    img {display: inline; height: auto; max-width: 100%%;}
                    pre {white-space: pre-wrap;}
                    iframe {width: 90vw; height: 50.625vw;} /* 16:9 */
                </style>
            </head>
            <body>%1$s</body>
            </html>"
        ]]>
    </string>
</resources>
{% endhighlight %}
</div>

### Performance

Using the above techniques to style `TextView` and `WebView` to display the sample HTML content yields the following performance statistics.

<div class="container">
    <div class="row">
        <div class="col-xs-10 col-xs-offset-1 col-md-8 col-md-offset-1">
            <a href="/assets/img/performance_textview.png"><img src="/assets/img/performance_textview.png" style="max-width:45%" /></a>
            <a href="/assets/img/performance_webview.png"><img src="/assets/img/performance_webview.png" style="max-width:45%" /></a>
        </div>
    </div>
</div>

<figcaption>Performance: TextView (left) vs WebView (right) (click for full size)</figcaption>

As seen from the performance monitor screenshots, `TextView` consumes significantly more memory than `WebView`, as it needs to hold bitmaps from all loaded images once they are fetched, regardless of whether they are visible on screen.

The example article has 7 images of various sizes with a combined file sizes of 2MB which would become bitmaps in memory. The amount of memory needed depends on how we sample or resize the fetched images, but all of them will need to be in memory at the same time regardless. If we have an article which holds an arbitrary number of images, we may run into the dreaded `OutOfMemoryException` very quickly. Thus for this use case, `TextView` is a clear no go.

On the other hand, `WebView` historically has been optimized to be memory efficient. Under the hood[^medium] (*highly recommended read!*), it loads content into tiles visible on screen and recycles them as we scroll, resulting in incredibly low memory profile. However, it needs to use more GPU and CPU to process and draw those tiles into pixels on the fly, probably explaining why it consumes more GPU and CPU than `TextView`.

So the trade-off here is between memory versus CPU & GPU consumption.

### Rendering

Applying above techniques to style `WebView` or handle images in `TextView` does not come for free. By 'preprocessing' our content for rendering, we inherently add certain inital delay to our user experience. A video is best demonstrates this point:

<iframe width="420" height="315" class="center-block video-portrait" src="https://www.youtube-nocookie.com/embed/greJFB4YJg4?rel=0" frameborder="0" allowfullscreen></iframe>

This initial delay may vary depending on devices. On high end devices it may not be noticeable, but we all know how many low end Android devices are around! Using a progress indicator when applicable would surely smoothen the experience.

Another side effect of `WebView` is that users may see some pixelated effects while scrolling as the tiles are recycled to render new content.

### Bottom line and gotchas
Both widgets have its highs and lows when it comes to rich text rendering. For arbitrary HTML, my choice would be to favor `WebView` over `TextView` for its low memory consumption and native support for HTML content. Basic HTML and CSS knowledge would be needed, but knowing them will benefit you anyway. If the HTML content is known to be limited to certain tags without images (e.g. forum posts), `TextView` would be a more sensible choice.

The article explores a simple layout where both widgets occupy the whole screen. Things may change in a much more complicated way when we place them in a layout hierarchy together with other widgets. Try it out and profile to see what works for you!

---
[^chiuki]: <http://chiuki.github.io/advanced-android-textview/>
[^flaurent]: <http://flavienlaurent.com/blog/2014/01/31/spans/>
[^medium]: <https://medium.com/@camaelon/what-s-in-a-web-browser-83793b51df6c>
[Materialistic]: https://github.com/hidroh/materialistic
[example]: http://www.propertyguru.com.sg/property-management-news/2015/11/111633/7-small-spaces-to-call-home
[Github]: https://github.com/hidroh/richtext
[TextView]: https://developer.android.com/reference/android/widget/TextView.html
[WebView]: https://developer.android.com/reference/android/webkit/WebView.html
[fromHtml]: https://developer.android.com/reference/android/text/Html.html#fromHtml(java.lang.String, android.text.Html.ImageGetter, android.text.Html.TagHandler)
[loadDataWithBaseURL]: https://developer.android.com/reference/android/webkit/WebView.html#loadDataWithBaseURL(java.lang.String, java.lang.String, java.lang.String, java.lang.String, java.lang.String)
[Spanned]: https://developer.android.com/reference/android/text/Spanned.html
[TextView tags]: https://commonsware.com/blog/Android/2010/05/26/html-tags-supported-by-textview.html
[ImageGetter]: https://developer.android.com/reference/android/text/Html.ImageGetter.html
[LinkMovementMethod]: https://developer.android.com/reference/android/text/method/LinkMovementMethod.html