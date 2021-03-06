# IflySdk
科大讯飞SDK，目前支持流式语音识别、语音合成。兼容Linux和Windows。

### 注意
（1）、其中的AppID和ApiKey为测试APP，每日只有500次调用量，用完即止。请更换为自己的APP。  
（2）、语音听写（流式版）请开启动态修正（仅中文普通话支持，免费），否则会运行出错！  
（3）、ASRRecordDemo（实时录音识别）和TTSPlayDemo（实时TTS播放）仅支持在Windows中运行，因为依赖于NAudio，而NAudio仅支持在Widnows上面录音和播放。其余demo均兼容Linux和Windows。同时欢迎Pull Linux demo。  

### 参考
- [语音听写（流式版）WebAPI C#Demo](http://bbs.xfyun.cn/forum.php?mod=viewthread&tid=42499&highlight=C%23)
- 官方TTS Demo
- [NAudio](https://github.com/naudio/NAudio "NAudio")

### 使用方法
#### ASR（语音听写）
语音识别有两种模式。第一种是识别一个完整的音频文件，第二种是实时识别，可以一边输入音频，一边进行语音识别。  
实时识别需要注意的是：a.整个会话时长最多持续60s，或者超过10s未发送数据服务器会自动断开连接。但是识别过程不会停止，会自动开启一个新的会话进行识别。b.每次输入的分片大小建议在3000以上，小于1000就可能影响到识别速率（也取决于输入频率）。

（1）、实时识别
```csharp
static async void ASR()
{
    string path = @"02.pcm";  //测试文件路径,自己修改
    int frameSize = 3200;
    byte[] data = File.ReadAllBytes(path);

    try
    {
        ASRApi iat = new ApiBuilder()
            .WithAppSettings(new AppSettings()
            {
                ApiKey = "7b845bf729c3eeb97be6de4d29e0b446",
                ApiSecret = "50c591a9cde3b1ce14d201db9d793b01",
                AppID = "5c56f257"
            })
            .UseError((sender, e) =>
            {
                Console.WriteLine("错误：" + e.Message);
            })
            .UseMessage((sender, e) =>
            {
                Console.WriteLine("实时结果：" + e);
            })
            .BuildASR();

        //计算识别时间
        Stopwatch sw = new Stopwatch();
        sw.Start();

        for (int i = 0; i < data.Length; i += frameSize)
        {
            //模拟说话暂停
            await Task.Delay(100);
            iat.Convert(SubArray(data, i, frameSize));
        }
        //结束本次会话
        iat.Stop();
        //等待本次会话结束
        while (iat.Status != ServiceStatus.Stopped)
        {
            await Task.Delay(10);
        }
        sw.Stop();
        Console.WriteLine($"总共花费{Math.Round(sw.Elapsed.TotalSeconds, 2)}秒。");
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
    }
}
```

（2）、识别一个完整的音频
```csharp
static async void ASRAudio()
{
    string path = @"02.pcm";  //测试文件路径,自己修改
    byte[] data = File.ReadAllBytes(path);

    try
    {
        ASRApi iat = new ApiBuilder()
            .WithAppSettings(new AppSettings()
            {
                ApiKey = "7b845bf729c3eeb97be6de4d29e0b446",
                ApiSecret = "50c591a9cde3b1ce14d201db9d793b01",
                AppID = "5c56f257"
            })
            .UseError((sender, e) =>
            {
                Console.WriteLine("错误：" + e.Message);
            })
            .UseMessage((sender, e) =>
            {
                Console.WriteLine("实时结果：" + e);
            })
            .BuildASR();

        //计算识别时间
        Stopwatch sw = new Stopwatch();
        sw.Start();

        ResultModel<string> result = await iat.ConvertAudio(data);
        if (result.Code == ResultCode.Success)
        {
            Console.WriteLine("\n识别结果：" + result.Data);
        }
        else
        {
            Console.WriteLine("\n识别错误：" + result.Message);
        }

        sw.Stop();
        Console.WriteLine($"总共花费{Math.Round(sw.Elapsed.TotalSeconds, 2)}秒。");
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
    }
}
```



#### TTS（语音合成）
TTS返回的实时结果为Base64编码的音频数据，需要自行解码。返回的最终结果为byte[]数组，可以直接保存为pcm音频。保存的音频不能直接打开播放，需要播放测试可以使用工具Audacity或FFmpeg。

```csharp
static async void TTS()
{
    string str = @"两只黄鹂鸣翠柳，一行白鹭上青天";
    try
    {
        TTSApi tts = new ApiBuilder()
            .WithAppSettings(new AppSettings()
            {
                ApiKey = "7b845bf729c3eeb97be6de4d29e0b446",
                ApiSecret = "50c591a9cde3b1ce14d201db9d793b01",
                AppID = "5c56f257"
            })
            .UseError((sender, e) =>
            {
                Console.WriteLine(e.Message);
            })
            .UseMessage((sender, e) =>
            {
                Console.WriteLine("结果：" + e.Substring(0, 20) + "...");  //Base64的结果。没显示完
            })
            .BuildTTS();

        ResultModel<byte[]> result = await tts.Convert(str);
        if (result.Code == ResultCode.Success)
        {
            //注意：转换后的结果为16K的单声道原始音频，可以使用ffmpeg来测速播放。
            string path = Path.Combine(Environment.CurrentDirectory, "test.pcm");
            using (var fs = new FileStream(path, FileMode.Create, FileAccess.Write))
            {
                fs.Write(result.Data, 0, result.Data.Length);
                fs.Flush();
            }
            if (File.Exists(path))
            {
                Console.WriteLine("保存成功！");
            }
            else
            {
                Console.WriteLine("保存失败！");
            }
        }
        else
        {
            Console.WriteLine("\n错误：" + result.Message);
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
    }
}
```
用FFmpeg播放合成的音频
```shell
ffplay -ar 16000 -channels 1 -f s16le -i test.pcm
```