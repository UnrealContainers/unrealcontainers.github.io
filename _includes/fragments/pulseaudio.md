If you want to enable audio support for Unreal Engine containers running on the {{ include.host }} then you will need to install the PulseAudio server from the system package repositories for your chosen Linux distribution. For example, under Debian or Ubuntu you would use the following command:

{% highlight bash %}
sudo apt-get install pulseaudio
{% endhighlight %}

When running inside a virtual machine that does not have access to any physical audio devices, the PulseAudio server will automatically create a dummy output audio device when it starts. By default, the dummy output device will use different audio parameters to those used by the Unreal Engine under Linux, which may lead to audio distortion issues. To prevent issues when using the dummy device with the Unreal Engine, you will need to add the following lines to the `/etc/pulse/daemon.conf` configuration file to override the default audio parameters to match those used by the Unreal Engine:

{% highlight bash %}
default-sample-format = float32le
default-sample-rate = 48000
default-sample-channels = 6
{% endhighlight %}

Once the PulseAudio server is installed and correctly configured, you can enable audio support inside Unreal Engine containers by bind-mounting the PulseAudio socket from the host system. See the [cloud rendering](../use-cases/cloud-rendering) page for further details on enabling audio support.
