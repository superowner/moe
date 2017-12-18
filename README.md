//original by coding2233
using UnityEngine;
using System.IO;
using System;
using NAudio;
using NAudio.Wave;
using System.Runtime.InteropServices;

namespace Imouto
{
    public class NAudioPlayer
    {
        ///<summary>
        /// 从文件读取 Stream
        /// </summary>
        public byte[] GetBytesFormFile(string fileName)
        {
            // 打开文件
            FileStream fileStream = new FileStream(fileName, FileMode.Open, FileAccess.Read, FileShare.Read);
            // 读取文件的 byte[]
            byte[] bytes = new byte[fileStream.Length];
            fileStream.Read(bytes, 0, bytes.Length);
            fileStream.Dispose();
            fileStream.Close();
            return bytes;
        }


        //http://gad.qq.com/article/detail/7182188
        #region ogg字节转AudioClip
        NVorbis.VorbisReader vorbis;
        public AudioClip FromOggData(byte[] data,string audioClipName= "ogg clip")
        {
            // Load the data into a stream
            MemoryStream oggstream = new MemoryStream(data);
       
            vorbis = new NVorbis.VorbisReader(oggstream, false);
      
            int samplecount = (int)(vorbis.SampleRate * vorbis.TotalTime.TotalSeconds);

            //  AudioClip audioClip = AudioClip.Create("clip", samplecount, vorbis.Channels, vorbis.SampleRate, false, true, OnAudioRead, OnAudioSetPosition);
            AudioClip audioClip= AudioClip.Create(audioClipName, samplecount, vorbis.Channels, vorbis.SampleRate, false, OnAudioRead);
            oggstream.Dispose();
            oggstream.Close();

            // Return the clip
            return audioClip;
        }
        void OnAudioRead(float[] data)
        {
            var f = new float[data.Length];
            vorbis.ReadSamples(f, 0, data.Length);
            for (int i = 0; i < data.Length; i++)
            {
                data[i] = f[i];
            }
        }
        void OnAudioSetPosition(int newPosition)
        {
            vorbis.DecodedTime = new TimeSpan(newPosition); //Only used to rewind the stream, we won't be seeking
        }
        #endregion

        public AudioClip FromWavData(byte[] data , string audioClipName = "wav clip")
        {
            //// Load the data into a stream
            //MemoryStream wavstream = new MemoryStream(data);
            //WaveFileReader wavaudio = new WaveFileReader(wavstream);
            //WaveStream waveStream = WaveFormatConversionStream.CreatePcmStream(wavaudio);
            //// Convert to WAV data
            //WAV wav = new WAV(AudioMemStream(waveStream).ToArray());

            WAV wav = new WAV(data);
            //Debug.Log(wav);
            AudioClip audioClip = AudioClip.Create(audioClipName, wav.SampleCount, 1, wav.Frequency, false);
            audioClip.SetData(wav.LeftChannel, 0);
            //wav
            // Return the clip
            return audioClip;
        }


        public WAV FromMp3Data(byte[] data)
        {
            System.Diagnostics.Stopwatch _Stopwatch = new System.Diagnostics.Stopwatch();
            _Stopwatch.Start();
            // Load the data into a stream
            MemoryStream mp3stream = new MemoryStream(data);
            // Convert the data in the stream to WAV format
            Mp3FileReader mp3audio = new Mp3FileReader(mp3stream);

            WaveStream waveStream = WaveFormatConversionStream.CreatePcmStream(mp3audio);

            //return null;
            //_Stopwatch.Stop();
            //Debug.Log("Load Finished,Costing:" + _Stopwatch.ElapsedMilliseconds / 1000f + "s");// Costing:0.018s

            var memoryStream = AudioMemStream(waveStream);//Costing:1.497s

            _Stopwatch.Stop();
            Debug.Log("Load Finished,Costing:" + _Stopwatch.ElapsedMilliseconds / 1000f + "s");
            _Stopwatch.Restart();

            return null;

            var array = memoryStream.ToArray();
          

            // Convert to WAV data
            var wav = new WAV(array);//Costing:0.641s
            _Stopwatch.Stop();
            Debug.Log("Load Finished,Costing:" + _Stopwatch.ElapsedMilliseconds / 1000f + "s");

            mp3stream.Dispose();
            mp3stream.Close();
            mp3audio.Dispose();
            mp3audio.Close();
            waveStream.Dispose();
            waveStream.Close();
            memoryStream.Dispose();
            memoryStream.Close();
            array = null;
            GC.Collect();
            return wav;
        }

        public AudioClip WavToAudioClip(WAV wav, string audioClipName = "mp3 clip")
        {
            AudioClip audioClip=null;
            try
            {
                audioClip = AudioClip.Create(audioClipName, wav.SampleCount, 1, wav.Frequency, false);
                audioClip.SetData(wav.LeftChannel, 0);
            }
            catch(Exception e)
            {
                Debug.Log("audioClipName:"+ audioClipName + "-->" + e);
            }

            return audioClip;
        }

        private MemoryStream AudioMemStream(WaveStream waveStream)
        {
            //MemoryStream outputStream = new MemoryStream();
            using (MemoryStream outputStream = new MemoryStream())
            {
                using (WaveFileWriter waveFileWriter = new WaveFileWriter(outputStream, waveStream.WaveFormat))
                {
                    byte[] bytes = new byte[waveStream.Length];
                    waveStream.Position = 0;
                    waveStream.Read(bytes, 0, Convert.ToInt32(waveStream.Length));
                    waveFileWriter.Write(bytes, 0, bytes.Length);
                    waveFileWriter.Flush();
                }
                return outputStream;
            }
      
        }
    }

    [StructLayout(LayoutKind.Sequential, Pack = 1)]
    public struct SimpleWAV
    {
        // properties
        [MarshalAs(UnmanagedType.ByValArray, SizeConst = 113091840)]
        public float[] LeftChannel;
        [MarshalAs(UnmanagedType.ByValArray, SizeConst = 113091840)]
        public float[] RightChannel;
        public int ChannelCount { get; internal set; }
        public int SampleCount { get; internal set; }
        public int Frequency { get; internal set; }

        //public override string ToString()
        //{
        //    return string.Format("[WAV: LeftChannel={0}, RightChannel={1}, ChannelCount={2}, SampleCount={3}, Frequency={4}]", LeftChannel, RightChannel, ChannelCount, SampleCount, Frequency);
        //}
    }

    /* From http://answers.unity3d.com/questions/737002/wav-byte-to-audioclip.html */
    //[StructLayout(LayoutKind.Sequential)]
    //[Serializable]
    public class WAV
    {

        // convert two bytes to one float in the range -1 to 1
        float bytesToFloat(byte firstByte, byte secondByte)
        {
            // convert two bytes to one short (little endian)
            short s = (short)((secondByte << 8) | firstByte);
            // convert to range from -1 to (just below) 1
            return s / 32768.0F;
        }

        int bytesToInt(byte[] bytes, int offset = 0)
        {
            int value = 0;
            for (int i = 0; i < 4; i++)
            {
                value |= ((int)bytes[offset + i]) << (i * 8);
            }
            return value;
        }
        // properties
        public float[] LeftChannel { get; internal set; }
        public float[] RightChannel { get; internal set; }
        public int ChannelCount { get; internal set; }
        public int SampleCount { get; internal set; }
        public int Frequency { get; internal set; }

        public WAV(byte[] wav)
        {
            //return; //535M
            // Determine if mono or stereo
            ChannelCount = wav[22];     // Forget byte 23 as 99.999% of WAVs are 1 or 2 channels
            //return; //
            // Get the frequency
            Frequency = bytesToInt(wav, 24);

            // Get past all the other sub chunks to get to the data subchunk:
            int pos = 12;   // First Subchunk ID from 12 to 16

            
            // Keep iterating until we find the data chunk (i.e. 64 61 74 61 ...... (i.e. 100 97 116 97 in decimal))
            while (!(wav[pos] == 100 && wav[pos + 1] == 97 && wav[pos + 2] == 116 && wav[pos + 3] == 97))
            {
                pos += 4;
                int chunkSize = wav[pos] + wav[pos + 1] * 256 + wav[pos + 2] * 65536 + wav[pos + 3] * 16777216;
                pos += 4 + chunkSize;
            }
            pos += 8;

            // Pos is now positioned to start of actual sound data.
            SampleCount = (wav.Length - pos) / 2;     // 2 bytes per sample (16 bit sound mono)
            if (ChannelCount == 2) SampleCount /= 2;        // 4 bytes per sample (16 bit stereo)

            // Allocate memory (right will be null if only mono sound)
            LeftChannel = new float[SampleCount];
            if (ChannelCount == 2) RightChannel = new float[SampleCount];
            else RightChannel = null;

            // Write to double array/s:
            int i = 0;
            int maxInput = wav.Length - (RightChannel == null ? 1 : 3);
            // while (pos < wav.Length)
            while ((i<SampleCount)&&( pos < maxInput))
            {
                LeftChannel[i] = bytesToFloat(wav[pos], wav[pos + 1]);
                pos += 2;
                if (ChannelCount == 2)
                {
                    RightChannel[i] = bytesToFloat(wav[pos], wav[pos + 1]);
                    pos += 2;
                }
                i++;
            }
        }

        public override string ToString()
        {
            return string.Format("[WAV: LeftChannel={0}, RightChannel={1}, ChannelCount={2}, SampleCount={3}, Frequency={4}]", LeftChannel, RightChannel, ChannelCount, SampleCount, Frequency);
        }
    }
}

