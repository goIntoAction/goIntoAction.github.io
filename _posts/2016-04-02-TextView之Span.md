---
layout:     post
title:      "TextView之span"
subtitle:   ""
date:       2016-05-29
author:     "goIntoAction"
header-img: "img/post-bg.jpg"
tags:
    - android源码分析
---

Span是TextView实现简单富文本效果的辅助类。常见的Span类有ImageSpan、URLSpan、TypefaceSpan等，分别实现显示图片和表情、超链接、不同字体。一段文字可以有一个或者多个Span。
文本的Span需要一个统一的管理者，这个就是Spanned。Spanned的是一个接口，定义来了获取Span的相关方法。Spannable是Spanned的子接口，主要定义了修改文本Span的方法如：setSpan、removeSpan。TextView中最常用的是其两个子类SpannableStringBuilder、和SpannableString。SpannableStringBuilder支持可编辑文本，SpannableString支持不可编辑文本。
SpannableString和SpannableStringBuilder的基本类图：<br/>
![span1.png](/img/in-post/span/span1.png)

如上图所示，SpannableStringBuilder继承了Editable所以可以编辑，SpannableString和SpannableStringBuilder都继承了CharSequence，所以可以直接传给TextView显示。

Span的显示过程
TextView的文本绘制主要是由在Layout中实现的，下面以ImageSpan为例，来看看Layout是如何处理Span的。
ImageSpan的类关系图：<br/>
![span2.png](/img/in-post/span/span2.png)


Layout绘制文本过程：
Layout.draw->Layout.drawText

    public void drawText(Canvas canvas, int firstLine, int lastLine) {
    ......

    TextLine tl = TextLine.obtain();
    ......
        if (directions == DIRS_ALL_LEFT_TO_RIGHT && !mSpannedText && !hasTabOrEmoji) {
            // XXX: assumes there's nothing additional to be done
            canvas.drawText(buf, start, end, x, lbaseline, paint);
        } else {
            tl.set(paint, buf, start, end, dir, directions, hasTabOrEmoji, tabStops);
            tl.draw(canvas, x, ltop, lbaseline, lbottom);
        }
    ......
    }
上面是Layout.drawText的关键代码，在方法最后会判断文本在有Span的情况下，使用TextLine去绘制文本。TextLine首先调用draw，按文本方向，是否是emoji表情分为不同的片段，在再调用drawRun，drawRun调用handleRun绘制每个文本片段，然后根据片段方向确定文本最后的边界位置。
TextLine.draw->TextLine.drawRun->TextLine.handleRun, handleRun是处理Span的核心方法，下面详细分析一下：

    private float handleRun(int start, int measureLimit,
            int limit, boolean runIsRtl, Canvas c, float x, int top, int y,
            int bottom, FontMetricsInt fmi, boolean needWidth) {

            // Case of an empty line, make sure we update fmi according to mPaint
            if (start == measureLimit) {
                TextPaint wp = mWorkPaint;
                wp.set(mPaint);
                if (fmi != null) {
                    expandMetricsFromPaint(fmi, wp);
                }
                return 0f;
            }

            if (mSpanned == null) { //如果没有Span，直接调用handleText绘制文本
                TextPaint wp = mWorkPaint;
                wp.set(mPaint);
                final int mlimit = measureLimit;
                return handleText(wp, start, mlimit, start, limit, runIsRtl, c, x, top,
                        y, bottom, fmi, needWidth || mlimit < measureLimit);
            }

            //初始化MetricAffectingSpanSpanSet，改变文本尺寸的Span容器。此类Span会原来文明的尺寸或者代替原来的文本，绘制Span定制的类容，如表情、图片、字体、文本缩放等。
        mMetricAffectingSpanSpanSet.init(mSpanned, mStart + start, mStart + limit);
        //初始化CharacterStyleSpanSet，改变文本样式的Span容器。此类Span通过修改TextPaint的属性实现特定效果，如字体颜色，下划线等。
        mCharacterStyleSpanSet.init(mSpanned, mStart + start, mStart + limit);

        // Shaping needs to take into account context up to metric boundaries,
        // but rendering needs to take into account character style boundaries.
        // So we iterate through metric runs to get metric bounds,
        // then within each metric run iterate through character style runs
        // for the run bounds.
        final float originalX = x;
        for (int i = start, inext; i < measureLimit; i = inext) {
            TextPaint wp = mWorkPaint;
            wp.set(mPaint);

            //查找下一个MetricAffectingSpan类型的Span
            inext = mMetricAffectingSpanSpanSet.getNextTransition(mStart + i, mStart + limit) -
                    mStart;
            int mlimit = Math.min(inext, measureLimit);

            ReplacementSpan replacement = null;

            for (int j = 0; j < mMetricAffectingSpanSpanSet.numberOfSpans; j++) {
                // Both intervals [spanStarts..spanEnds] and [mStart + i..mStart + mlimit] are NOT
                // empty by construction. This special case in getSpans() explains the >= & <= tests
                //找到符合片段位置的Span。
                if ((mMetricAffectingSpanSpanSet.spanStarts[j] >= mStart + mlimit) ||
                        (mMetricAffectingSpanSpanSet.spanEnds[j] <= mStart + i)) continue;
                MetricAffectingSpan span = mMetricAffectingSpanSpanSet.spans[j];

                //如果是ReplacementSpan类型的Span，赋值给replacement，否则调用Span的updateDrawState方法跟新TextPaint状态。
                if (span instanceof ReplacementSpan) {
                    replacement = (ReplacementSpan)span;
                } else {
                    // We might have a replacement that uses the draw
                    // state, otherwise measure state would suffice.
                    span.updateDrawState(wp);
                }
            }

            //如果是ReplacementSpan类型的Span，调用handleReplacement方法绘制Span内容。
            if (replacement != null) {
                x += handleReplacement(replacement, wp, i, mlimit, runIsRtl, c, x, top, y,
                        bottom, fmi, needWidth || mlimit < measureLimit);
                continue;
            }

            for (int j = i, jnext; j < mlimit; j = jnext) {
                //找到符合片段位置的Span。
                jnext = mCharacterStyleSpanSet.getNextTransition(mStart + j, mStart + mlimit) -
                        mStart;

                wp.set(mPaint);
                for (int k = 0; k < mCharacterStyleSpanSet.numberOfSpans; k++) {
                    // Intentionally using >= and <= as explained above
                    if ((mCharacterStyleSpanSet.spanStarts[k] >= mStart + jnext) ||
                            (mCharacterStyleSpanSet.spanEnds[k] <= mStart + j)) continue;

                    CharacterStyle span = mCharacterStyleSpanSet.spans[k];
                    //调用Span的updateDrawState方法跟新TextPaint状态。
                    span.updateDrawState(wp);
                }
                //绘制Span范围内的文本
                x += handleText(wp, j, jnext, i, inext, runIsRtl, c, x,
                        top, y, bottom, fmi, needWidth || jnext < measureLimit);
            }
        }

        return x - originalX;
	}
    
    
ImageSpan是ReplacementSpan的子类，根据上面的代码分析，ImageSpan最终是在TextLine. handleReplacement方法中绘制的。handleReplacement的处理很简单，如果是从右向左的方向或者需要返回最后的边界，需要计算ReplacementSpan的宽度，通过ReplacementSpan.getSize方法计算宽度，ImageSpan这里就是图片的宽度。最后调用DynamicDrawableSpan.draw方法绘制图片。

	private float handleReplacement(ReplacementSpan replacement, TextPaint wp,
        int start, int limit, boolean runIsRtl, Canvas c,
        float x, int top, int y, int bottom, FontMetricsInt fmi,
        boolean needWidth) {

        float ret = 0;

        int textStart = mStart + start;
        int textLimit = mStart + limit;

        //需要返回文本宽度或者文本方向是从右到左，需要计算Span的宽度。
        if (needWidth || (c != null && runIsRtl)) {
            int previousTop = 0;
            int previousAscent = 0;
            int previousDescent = 0;
            int previousBottom = 0;
            int previousLeading = 0;

            boolean needUpdateMetrics = (fmi != null);

            if (needUpdateMetrics) {
                previousTop     = fmi.top;
                previousAscent  = fmi.ascent;
                previousDescent = fmi.descent;
                previousBottom  = fmi.bottom;
                previousLeading = fmi.leading;
            }

            ret = replacement.getSize(wp, mText, textStart, textLimit, fmi);

            if (needUpdateMetrics) {
                updateMetrics(fmi, previousTop, previousAscent, previousDescent, previousBottom,
                        previousLeading);
            }
        }

        if (c != null) {
            //如果文本是从右到左，起始位置是x坐标减去Span的宽度
            if (runIsRtl) {
                x -= ret;
            }
            //绘制Span
            replacement.draw(c, mText, textStart, textLimit,
                    x, top, y, bottom, wp);
        }

        return runIsRtl ? -ret : ret;
	}
    
URLSpan
URLSpan是很非常常用的一种Span。它的主要功能是在文本中突出显示超链接，点击链接跳转到对应的处理应用。
<br/>
![span3.png](/img/in-post/span/span3.png)

从上面的类图可以看出来，URLSpan是ClickableSpan的子类，也是CharacterStyle的子类，将会继承updateDrawState和onClick方法。updateDrawState可以修改TextPaint的属性，实现修改字体颜色、下划线等效果，onClick可以实现点击效果，处理点击动作。
TextView要实现超链接的效果，可以利用SpannableString或者SpannableStringBuilder构建字符串，在超链接的位置设置URLSpan。
TextView也可以开启自动识别超链接的功能。通过调用setAutoLinkMask(int mask)，可以让TextView自动识别电子邮件、电话号码、网址、地图位置中的一个、多个或者全部链接。mast参数用来控制识别行为，对应的值定义在Linkify辅助类中。
public static final int WEB_URLS = 0x01;  //网址

public static final int EMAIL_ADDRESSES = 0x02;  //邮件地址

public static final int PHONE_NUMBERS = 0x04;  //电话号码

public static final int MAP_ADDRESSES = 0x08;  //地图位置

public static final int ALL = WEB_URLS | EMAIL_ADDRESSES | PHONE_NUMBERS | MAP_ADDRESSES;  //网址、邮件地址、电话号码、地图位置都识别。
下面分析一下TextView是如何自动识别超链接，如何把点击事件传递给URLSpan处理的。TextView设置文本会调用private void setText(CharSequence text, BufferType type, boolean notifyBefore, int oldlen)，此方法会对mAutoLinkMask处理，关键代码如下：

	//如果设置的mAutoLinkMask
    if (mAutoLinkMask != 0) {
        Spannable s2;

        if (type == BufferType.EDITABLE || text instanceof Spannable) {
            s2 = (Spannable) text;
        } else {
            s2 = mSpannableFactory.newSpannable(text);
        }

        if (Linkify.addLinks(s2, mAutoLinkMask)) {  //调用Linkify.addLinks识别链接、设置URLSpan
            text = s2;
            type = (type == BufferType.EDITABLE) ? BufferType.EDITABLE : BufferType.SPANNABLE;

            /*
             * We must go ahead and set the text before changing the
             * movement method, because setMovementMethod() may call
             * setText() again to try to upgrade the buffer type.
             */
            mText = text;

            // Do not change the movement method for text that support text selection as it
            // would prevent an arbitrary cursor displacement.
            if (mLinksClickable && !textCanBeSelected()) {
                //设置LinkMovementMethod，处理超链接点击事件
                setMovementMethod(LinkMovementMethod.getInstance());
            }
        }
    }
    
上面代码中，Linkify.addLinks是识别超链接设置URLSpan方法。LinkMovementMethod是处理超链接点击的类。
TextView在处理触摸事件时，会把事件出递给MovementMethod处理，这里就是LinkMovementMethod。

	public boolean onTouchEvent(MotionEvent event) {
    ......

        final boolean touchIsFinished = (action == MotionEvent.ACTION_UP) &&
                (mEditor == null || !mEditor.mIgnoreActionUpEvent) && isFocused();

         if ((mMovement != null || onCheckIsTextEditor()) && isEnabled()
                && mText instanceof Spannable && mLayout != null) {
            boolean handled = false;

            if (mMovement != null) {
                //把MotionEvent传递给MovementMethod处理
                handled |= mMovement.onTouchEvent(this, (Spannable) mText, event);
            }

    ......

        return superResult;
	}
    
LinkMovementMethod. onTouchEvent中只处理ACTION_UP和ACTION_DOWN。首先计算出点击文本的位置，再查找该位置上是否有可点解的Span，如果有，在ACTION_DOWN时就选中该段文本，在ACTION_UP时就调用Span的onClick方法。
下面是详细代码分析：

	public boolean onTouchEvent(TextView widget, Spannable buffer,
                            MotionEvent event) {
        int action = event.getAction();

        if (action == MotionEvent.ACTION_UP ||
            action == MotionEvent.ACTION_DOWN) {
            int x = (int) event.getX();
            int y = (int) event.getY();

            x -= widget.getTotalPaddingLeft();
            y -= widget.getTotalPaddingTop();

            x += widget.getScrollX();
            y += widget.getScrollY();

            Layout layout = widget.getLayout();
            int line = layout.getLineForVertical(y);
            int off = layout.getOffsetForHorizontal(line, x);
            //根据点击位置查找可点击的Span
            ClickableSpan[] link = buffer.getSpans(off, off, ClickableSpan.class);

            if (link.length != 0) {
                if (action == MotionEvent.ACTION_UP) {
                    //如果是ACTION_UP事件就调用Span的onClick方法处理点击事件。
                    link[0].onClick(widget);
                } else if (action == MotionEvent.ACTION_DOWN) {
                    //如果是ACTION_DOWN就是选中Span范围类的文字。
                    Selection.setSelection(buffer,
                                           buffer.getSpanStart(link[0]),
                                           buffer.getSpanEnd(link[0]));
                }

                return true;
            } else {
                Selection.removeSelection(buffer);
            }
        }

        return super.onTouchEvent(widget, buffer, event);
    }
