# Playback tutorial 3: Short-cutting the pipeline

# Goal

[Basic tutorial 8: Short-cutting the
pipeline](Basic%2Btutorial%2B8%253A%2BShort-cutting%2Bthe%2Bpipeline.html) showed
how an application can manually extract or inject data into a pipeline
by using two special elements called `appsrc` and `appsink`.
`playbin` allows using these elements too, but the method to connect
them is different. To connect an `appsink` to `playbin` see [Playback
tutorial 7: Custom playbin
sinks](Playback%2Btutorial%2B7%253A%2BCustom%2Bplaybin%2Bsinks.html).
This tutorial shows:

  - How to connect `appsrc` with `playbin`
  - How to configure the `appsrc`

# A playbin waveform generator

Copy this code into a text file named `playback-tutorial-3.c`.

<table>
<tbody>
<tr class="odd">
<td><img src="images/icons/emoticons/information.png" width="16" height="16" /></td>
<td><p>This tutorial is included in the SDK since release 2012.7. If you cannot find it in the downloaded code, please install the latest release of the GStreamer SDK.</p></td>
</tr>
</tbody>
</table>

**playback-tutorial-3.c**

``` c
#include <gst/gst.h>
#include <string.h>

#define CHUNK_SIZE 1024   /* Amount of bytes we are sending in each buffer */
#define SAMPLE_RATE 44100 /* Samples per second we are sending */
#define AUDIO_CAPS "audio/x-raw-int,channels=1,rate=%d,signed=(boolean)true,width=16,depth=16,endianness=BYTE_ORDER"

/* Structure to contain all our information, so we can pass it to callbacks */
typedef struct _CustomData {
  GstElement *pipeline;
  GstElement *app_source;

  guint64 num_samples;   /* Number of samples generated so far (for timestamp generation) */
  gfloat a, b, c, d;     /* For waveform generation */

  guint sourceid;        /* To control the GSource */

  GMainLoop *main_loop;  /* GLib's Main Loop */
} CustomData;

/* This method is called by the idle GSource in the mainloop, to feed CHUNK_SIZE bytes into appsrc.
 * The ide handler is added to the mainloop when appsrc requests us to start sending data (need-data signal)
 * and is removed when appsrc has enough data (enough-data signal).
 */
static gboolean push_data (CustomData *data) {
  GstBuffer *buffer;
  GstFlowReturn ret;
  int i;
  gint16 *raw;
  gint num_samples = CHUNK_SIZE / 2; /* Because each sample is 16 bits */
  gfloat freq;

  /* Create a new empty buffer */
  buffer = gst_buffer_new_and_alloc (CHUNK_SIZE);

  /* Set its timestamp and duration */
  GST_BUFFER_TIMESTAMP (buffer) = gst_util_uint64_scale (data->num_samples, GST_SECOND, SAMPLE_RATE);
  GST_BUFFER_DURATION (buffer) = gst_util_uint64_scale (CHUNK_SIZE, GST_SECOND, SAMPLE_RATE);

  /* Generate some psychodelic waveforms */
  raw = (gint16 *)GST_BUFFER_DATA (buffer);
  data->c += data->d;
  data->d -= data->c / 1000;
  freq = 1100 + 1000 * data->d;
  for (i = 0; i < num_samples; i++) {
    data->a += data->b;
    data->b -= data->a / freq;
    raw[i] = (gint16)(500 * data->a);
  }
  data->num_samples += num_samples;

  /* Push the buffer into the appsrc */
  g_signal_emit_by_name (data->app_source, "push-buffer", buffer, &ret);

  /* Free the buffer now that we are done with it */
  gst_buffer_unref (buffer);

  if (ret != GST_FLOW_OK) {
    /* We got some error, stop sending data */
    return FALSE;
  }

  return TRUE;
}

/* This signal callback triggers when appsrc needs data. Here, we add an idle handler
 * to the mainloop to start pushing data into the appsrc */
static void start_feed (GstElement *source, guint size, CustomData *data) {
  if (data->sourceid == 0) {
    g_print ("Start feeding\n");
    data->sourceid = g_idle_add ((GSourceFunc) push_data, data);
  }
}

/* This callback triggers when appsrc has enough data and we can stop sending.
 * We remove the idle handler from the mainloop */
static void stop_feed (GstElement *source, CustomData *data) {
  if (data->sourceid != 0) {
    g_print ("Stop feeding\n");
    g_source_remove (data->sourceid);
    data->sourceid = 0;
  }
}

/* This function is called when an error message is posted on the bus */
static void error_cb (GstBus *bus, GstMessage *msg, CustomData *data) {
  GError *err;
  gchar *debug_info;

  /* Print error details on the screen */
  gst_message_parse_error (msg, &err, &debug_info);
  g_printerr ("Error received from element %s: %s\n", GST_OBJECT_NAME (msg->src), err->message);
  g_printerr ("Debugging information: %s\n", debug_info ? debug_info : "none");
  g_clear_error (&err);
  g_free (debug_info);

  g_main_loop_quit (data->main_loop);
}

/* This function is called when playbin has created the appsrc element, so we have
 * a chance to configure it. */
static void source_setup (GstElement *pipeline, GstElement *source, CustomData *data) {
  gchar *audio_caps_text;
  GstCaps *audio_caps;

  g_print ("Source has been created. Configuring.\n");
  data->app_source = source;

  /* Configure appsrc */
  audio_caps_text = g_strdup_printf (AUDIO_CAPS, SAMPLE_RATE);
  audio_caps = gst_caps_from_string (audio_caps_text);
  g_object_set (source, "caps", audio_caps, NULL);
  g_signal_connect (source, "need-data", G_CALLBACK (start_feed), data);
  g_signal_connect (source, "enough-data", G_CALLBACK (stop_feed), data);
  gst_caps_unref (audio_caps);
  g_free (audio_caps_text);
}

int main(int argc, char *argv[]) {
  CustomData data;
  GstBus *bus;

  /* Initialize cumstom data structure */
  memset (&data, 0, sizeof (data));
  data.b = 1; /* For waveform generation */
  data.d = 1;

  /* Initialize GStreamer */
  gst_init (&argc, &argv);

  /* Create the playbin element */
  data.pipeline = gst_parse_launch ("playbin uri=appsrc://", NULL);
  g_signal_connect (data.pipeline, "source-setup", G_CALLBACK (source_setup), &data);

  /* Instruct the bus to emit signals for each received message, and connect to the interesting signals */
  bus = gst_element_get_bus (data.pipeline);
  gst_bus_add_signal_watch (bus);
  g_signal_connect (G_OBJECT (bus), "message::error", (GCallback)error_cb, &data);
  gst_object_unref (bus);

  /* Start playing the pipeline */
  gst_element_set_state (data.pipeline, GST_STATE_PLAYING);

  /* Create a GLib Main Loop and set it to run */
  data.main_loop = g_main_loop_new (NULL, FALSE);
  g_main_loop_run (data.main_loop);

  /* Free resources */
  gst_element_set_state (data.pipeline, GST_STATE_NULL);
  gst_object_unref (data.pipeline);
  return 0;
}
```

To use an `appsrc` as the source for the pipeline, simply instantiate a
`playbin` and set its URI to `appsrc://`

``` c
/* Create the playbin element */
data.pipeline = gst_parse_launch ("playbin uri=appsrc://", NULL);
```

`playbin` will create an internal `appsrc` element and fire the
`source-setup` signal to allow the application to configure
it:

``` c
g_signal_connect (data.pipeline, "source-setup", G_CALLBACK (source_setup), &data);
```

In particular, it is important to set the caps property of `appsrc`,
since, once the signal handler returns, `playbin` will instantiate the
next element in the pipeline according to these
caps:

``` c
/* This function is called when playbin has created the appsrc element, so we have
 * a chance to configure it. */
static void source_setup (GstElement *pipeline, GstElement *source, CustomData *data) {
  gchar *audio_caps_text;
  GstCaps *audio_caps;

  g_print ("Source has been created. Configuring.\n");
  data->app_source = source;

  /* Configure appsrc */
  audio_caps_text = g_strdup_printf (AUDIO_CAPS, SAMPLE_RATE);
  audio_caps = gst_caps_from_string (audio_caps_text);
  g_object_set (source, "caps", audio_caps, NULL);
  g_signal_connect (source, "need-data", G_CALLBACK (start_feed), data);
  g_signal_connect (source, "enough-data", G_CALLBACK (stop_feed), data);
  gst_caps_unref (audio_caps);
  g_free (audio_caps_text);
}
```

The configuration of the `appsrc` is exactly the same as in [Basic
tutorial 8: Short-cutting the
pipeline](Basic%2Btutorial%2B8%253A%2BShort-cutting%2Bthe%2Bpipeline.html):
the caps are set to `audio/x-raw-int`, and two callbacks are registered,
so the element can tell the application when it needs to start and stop
pushing data. See [Basic tutorial 8: Short-cutting the
pipeline](Basic%2Btutorial%2B8%253A%2BShort-cutting%2Bthe%2Bpipeline.html)
for more details.

From this point onwards, `playbin` takes care of the rest of the
pipeline, and the application only needs to worry about generating more
data when told so.

To learn how data can be extracted from `playbin` using the
`appsink` element, see [Playback tutorial 7: Custom playbin
sinks](Playback%2Btutorial%2B7%253A%2BCustom%2Bplaybin%2Bsinks.html).

# Conclusion

This tutorial applies the concepts shown in [Basic tutorial 8:
Short-cutting the
pipeline](Basic%2Btutorial%2B8%253A%2BShort-cutting%2Bthe%2Bpipeline.html) to
`playbin`. In particular, it has shown:

  - How to connect `appsrc` with `playbin` using the special
    URI `appsrc://`
  - How to configure the `appsrc` using the `source-setup` signal

It has been a pleasure having you here, and see you soon\!

## Attachments:

![](images/icons/bullet_blue.gif)
[playback-tutorial-3.c](attachments/1442200/2424850.c) (text/plain)
![](images/icons/bullet_blue.gif)
[vs2010.zip](attachments/1442200/2424849.zip) (application/zip)
![](images/icons/bullet_blue.gif)
[playback-tutorial-3.c](attachments/1442200/2424848.c) (text/plain)