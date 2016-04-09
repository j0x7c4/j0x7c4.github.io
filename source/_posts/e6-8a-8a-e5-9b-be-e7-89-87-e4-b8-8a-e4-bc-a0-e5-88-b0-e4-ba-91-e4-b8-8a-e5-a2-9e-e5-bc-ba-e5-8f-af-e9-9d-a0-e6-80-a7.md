---
title: 把图片上传到云上——增强可靠性
tags:
  - jQuery
  - jQuery-File-Upload
id: 34
categories:
  - 前端工程师
date: 2015-11-18 16:51:45
---

用了几天腾讯的图片云之后发现上传图片时偶尔会有失败的情况，之前的代码没有对上传失败或者取消上传进行处理。

比较可靠的流程应该是：

1.  点击上传，显示取消上传的按钮;
2.  如果上传失败，或者点了取消上传的按钮，显示重传的按钮;
3.  如果一切正常，显示上传成功的提示。

将之前add事件中的data.context改成:
{% codeblock lang:javascript %}
    data.context = $('<button/>').text('上传').addClass("btn btn-primary")
    .appendTo($(".action-"+uid))
    .click(function () {
        var $this = $(this);
        $this
                .off('click')
                .text("取消上传").removeClass().addClass("btn btn-warning")
                .on('click', function () {
                    data.abort();
                });
        data.submit();
    });
{% endcodeblock %}

    当点了取消上传，或者上传失败后，会触发fail事件，再修改fail时间的函数：

{% codeblock lang:javascript %}
    fail:function (e, data) {
        var result = data.result;
        if (result) {
            result = JSON.parse(data.result);
        } else {
            result = data.response().textStatus;
        }
        alert("上传失败("+result+")");
        data.context.text("重传").removeClass().addClass("btn btn-danger").on('click', function () {
            $.getJSON("/doAction/picauthtest", function(ret) {
                var sign = encodeURIComponent(ret.data.sign);
                var url = ret.data.url + '?sign=' + sign;
                data.url = url;
                data.submit();
            });
        });
    }
{% endcodeblock %}

如果是腾讯图片云返回的错误信息，结果存在data.result中，是一个json字符串; 而如果是其他错误，需要从.response()方法获得。这里要注意的是在重传时要重新获取一次上传的url.

这样修改可以提高上传图片时的用户体验，虽然是随便写写前端的代码，但是考虑问题还是要周全一些。
