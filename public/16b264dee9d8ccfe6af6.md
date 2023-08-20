---
title: FFTでギターの演奏を正誤判定
tags:
  - C#
  - Unity
  - 音楽
  - 音声処理
  - Guitar
private: false
updated_at: '2020-12-26T11:41:59+09:00'
id: 16b264dee9d8ccfe6af6
organization_url_name: null
slide: false
---
#１行概要
マイクで拾った演奏音をもとに、**FFTで出力した周波数スペクトラムのピーク値の組み合わせ**から、演奏の正誤を判定する。

#はじめに
ギターの基本的な演奏技術の一つに、複数の弦を同時に指で押さえて和音を奏でる『コード弾き』があるのは周知の事実である。本稿はそんなコード弾きを独自に練習する際に役立つと考える、コード弾き演奏音から正誤判定を行う手法を実現する。
具体的には、マイクで演奏した音を拾い、コンピュータで周波数成分を検出、検出結果をもとに正誤の判定を行う。
なお、本稿で想定するシステムユースケースは、コード弾きのお手本をこちらから提示し予め演奏されるであろう音が分かっているというシナリオであり、演奏者がランダムに演奏する音のコード認識をするというものではない。

#処理の流れ
マイクで取得した音情報に対して次のような操作を行う。
![16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/488859/343955f0-ca29-1954-d204-8de0c424a222.png)

###1. フーリエ変換
フーリエ変換およびFFT（Fast Fourier Transform）についての詳細な説明は割愛するが、一言でいえば「周波数ごとの成分を数値的に見ることができるようにする変換」である。

今回開発フレームとして利用しているUnityには、`AudioSource.GetSpectrumData`というFFTスクリプトが既に用意されている。AudioSourceとして使用するマイクを指定し、FFT結果の周波数スペクトラムデータを受け取る2のべき乗サイズの配列を用意しておく。
以下図は実際に『Cメジャー』と呼ばれるコードを弾いた際の周波数スペクトラムである。

![図1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/488859/fc2de8bb-1364-87d0-250b-03c04dac5275.png)

###2. ピーク値の探索
フーリエ変換により算出された周波数スペクトラムにおいて、ピークが立っている周波数がすなわち演奏された音と想定できる。したがって、ピーク値を探索し、検出されたピーク値の組み合わせから演奏の正誤を判定することができる。

一般的な（6弦）ギターは、最低周波数が82Hz（6弦開放）のE2、最高周波数が1245Hz（1弦23f）のD#6であり、この範囲内でピークを探索する。また、音階間の幅はE2とF2の間の5Hzが最小で、音階が高くなるほど大きくなっていくため、ピークを探索する幅も5Hzごととする。

ピーク値としては5Hz区間で探索する最大値をそのまま採用するのではなく、最大値間の最小値も同時に探索し、算出された最大値と最小値の差が閾値以上の場合にのみピーク値として採用する。これにより、ピークが立っていない、すなわちなだらかな区間での最大値を外すことができる。

###3. 周波数を音階に変換
一般によく知られるドレミファソラシド、ギターをはじめとする国際スタンダードCDEFGAB。これらの音階は、1オクターブを分割した一つ一つ（という認識）であり、440Hz / ラ / Aを基音に、1オクターブ高くなると周波数が2倍になると定義されている。
ギターにおける音階は12音階であり、1オクターブの間にC、C#、D、D#、E、F、F#、G、G#、A、A#、Bが存在する。前述した通り、ギターの最低音はE2の82Hzであり、E3の164Hz、その間82Hzを12分割しているということである。

話は長くなったが、本稿では便宜的にC2の65.5Hzを0、C3の131Hzを12とし、`scale = 12.0f * Mathf.Log(hertz / 65.5f) / Mathf.Log(2.0f)`で周波数を数値的な音階に変換し、四捨五入を行ったのち変換した数値をもとに音階とオクターブをそれぞれ算出する。

###4. コードごとに正誤判定
あとは、変換した音階と演奏されるであろう音とを比較するだけである。

以下図は作例であるが、変換した音階を表示し、演奏されるであろう音と比較した結果に応じて、『人差指の位置が間違えています!』『GOOD!』などの判定を表示している。
![17.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/488859/0d321cf9-58a9-61fb-de8d-41daa9443508.jpeg)
![18.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/488859/81ae1bb8-522e-3ef8-f4a3-dfb9214c4c02.jpeg)


#コード例
```c#:Analyze.cs
// FFT分解能（2の累乗）
private static int FFT_RESOLUTION = 2048;
// 採用する最低周波域
private static int FREQUENCY_RANGE = 450;
// 極値を算出する幅
private static int EXTREMUM_RANGE = 5;
// 極小値-極大値の閾値
private static float EXTREMUM_THRESHOLD = 0.0005f;

// a ～ bでの最大値インデックスを返す
public static int GetMaxNo(int a, int b, float[] c)
{
    int maxNum = a;
    float max = c[a];
    for (int i = a; i < a + b && i < c.Length; i++)
    {
        if (c[i] > max)
        {
            max = c[i];
            maxNum = i;
        }
    }
    return maxNum;
}
// a ～ bでの最小値インデックスを返す
public static int GetMinNo(int a, int b, float[] c)
{
    int minNum = a;
    float min = c[a];
    for (int i = a; i < b; i++)
    {
        if (min > c[i])
        {
            min = c[i];
            minNum = i;
        }
    }
    return minNum;
}

// FFT，ピーク値の算出・リターン
public static float[] AnalyzeSound(AudioSource audio)
{
    // FFT
    float[] spectrum = new float[FFT_RESOLUTION];
    audio.GetSpectrumData(spectrum, 0, FFTWindow.BlackmanHarris);
    /*
     * 周波数スペクトル波形をSceneに描画【テスト用】
     *
     * for (int i = 1; i < spectrum.Length - 1; ++i)
     * {
     *     Debug.DrawLine(
     *             new Vector3(Mathf.Log(i - 1), Mathf.Log(spectrum[i - 1]), 3),
     *             new Vector3(Mathf.Log(i), Mathf.Log(spectrum[i]), 3),
     *             Color.yellow);
     * }
     */

    // 極大値インデックスの配列
    int[] maxes = new int[spectrum.Length];
    // 極小値インデックスの配列
    int[] mins = new int[spectrum.Length];
    // ピーク値インデックスの配列
    int[] peaks = new int[spectrum.Length];

    //極大値の探索
    int count = 0;
    for (int i = 0; i < spectrum.Length - EXTREMUM_RANGE; i++)
    {
        if (GetMaxNo(i, EXTREMUM_RANGE, spectrum) == GetMaxNo(i + 1, EXTREMUM_RANGE, spectrum))
        {
            int check = 0;
            for (int k = 1; k < EXTREMUM_RANGE; k++)
            {
                if (GetMaxNo(i, EXTREMUM_RANGE, spectrum) == GetMaxNo(i + k, EXTREMUM_RANGE, spectrum))
                {
                    check++;
                }
            }
            if (check == EXTREMUM_RANGE - 1)
            {
                maxes[count] = GetMaxNo(i, EXTREMUM_RANGE, spectrum);
                count++;
            }
        }   
    }

    //極小値の探索
    mins[0] = GetMinNo(0, maxes[0], spectrum);
    for (int i = 0; i < spectrum.Length; i++)
    {
        if (maxes[i + 1] == 0) break;
        mins[i + 1] = GetMinNo(maxes[i], maxes[i + 1], spectrum);
    }

    //差分の計算
    int peakscnt = 0;
    for (int i = 0; i < spectrum.Length; i++)
    {
        if (spectrum[maxes[i]] - spectrum[mins[i]] >= EXTREMUM_THRESHOLD)
        {
            peaks[peakscnt] = maxes[i];
            peakscnt++;
        }
    }

    // ピーク周波数を返す
    // ピーク周波数インデックスの配列
    float[] freqs = new float[peakscnt];
    // ピーク周波数
    float[] pitches = new float[peakscnt];
    // 各ピークの前後のスペクトルも考慮
    for (int i = 0; i < peakscnt; i++)
    {
        freqs[i] = peaks[i];
        if (peaks[i] > 0 && peaks[i] < spectrum.Length - 1)
        {
            float dL = spectrum[peaks[i] - 1] / spectrum[peaks[i]];
            float dR = spectrum[peaks[i] + 1] / spectrum[peaks[i]];

            freqs[i] += 0.5f * (dR * dR - dL * dL);
        }
        pitches[i] = freqs[i] * (AudioSettings.outputSampleRate / 2) / spectrum.Length;
        // 検出する周波数域を考慮
        if (pitches[i] >= FREQUENCY_RANGE) pitches[i] = 0;
    }
    return pitches;
}

// 周波数から音階に変換
public static string ConvertHertzToScale(float hertz)
{
    // 周波数を，C2を0，（中略），C3を12とする数値に変換
    float scale = 12.0f * Mathf.Log(hertz / 65.5f) / Mathf.Log(2.0f);
    // 四捨五入
    int s = (int)scale;
    if (scale - s >= 0.5) s += 1;

    int smod = s % 12; // 音階
    int soct = s / 12; // オクターブ

    // 音階
    string value;
    switch(smod)
    {
        case 0:
          value = "C";
          break;
        case 1:
          value = "C#";
          break;
        case 2:
          value = "D";
          break;
        case 3:
          value = "D#";
          break;
        case 4:
          value = "E";
          break;
        case 5:
          value = "F";
          break;
        case 6:
          value = "F#";
          break;
        case 7:
          value = "G";
          break;
        case 8:
          value = "G#";
          break;
        case 9:
          value = "A";
          break;
        case 10:
          value = "A#";
          break;
        case 11:
          value = "B";
          break;
        default:
          value = "EXCEPTION";
          break;
    }
    value += soct + 2;

    return value;
}

void Start () {
        // マイク入力
        Mic.clip = Microphone.Start(null, true, 999, 44100);
        while (!(Microphone.GetPosition(null) > 0)) { }
        Mic.Play();
}

void Update () {
        // FFT，ピーク値の算出・リターン
        float[] hertz = AnalyzeSound(Mic);
        // 各ピーク周波数に対して
        for (int i = 0; i < hertz.Length; i++)
        {
            if (hertz[i] == 0) break;
            // 周波数から音階に変換
            string scale = ConvertHertzToScale(hertz[i]);
            /* 演奏された音の周波数と音階を描画【テスト用】
             * Debug.Log(hertz[i] + "Hz, Scale:" + scale);
             */
        }
}
```

#おわりに
今回は、FFTで出力した周波数スペクトラムのピーク値の組み合わせから、演奏の正誤を判定する手法を提示した。
正直なところ、判定性能は満足のいくものではない。マイクの設置する位置や指向性、性能に依存することが主な要因である。シンセサイザを噛ませたり、電気信号的に判定するといったハードウェアによる改善は比較的容易であるが、できればシステマチックに解決する手法を模索してみたい。
