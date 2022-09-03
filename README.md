# Michael Javier - Logstash Exercise README 
## TLDR 

Please see the [stdout.txt](stdout.txt) for examples of the filter logstash output after filtering, and the query results of my Elasticsearch instance where Logstash was saving documents after it finished filtering logs.


## Input Section
``` 
input {
  # stdin {}
  file {
    # HAS TO BE ABSOLUTE PATH FOR... REASONS??????? 
    path => ["C:/DevTools/LogStash/logstash-8.4.0/data/cyderes-mwj/ingest.log"]
    mode => "tail" # This lets me re-run the test just by pasting a new line to the ingest.log
  }
  # beats { # Saving this for later.  I might use FileBeats for a little playlist app that I want to build.
  #   port => 5044
  # }
}
```
The input section was interesting.  

It took me a little while to get the `file` plugin to actually read the `injest.log` file.  At first, I had `file.mode` set to `read`, but I got tired of having to edit the `mwj-logstash.conf`even though all I wanted to do was just add another line to the `ingest.log`.

When I set this up the first time (before I got the logstash exercise), I was just doing everything from `stdin`.  I was bummed that I had to stop using stdin because apparently, `--config-reload.automatic` doesn't support it.

I'm excited to look more into the various `beats` plugins.  I want to make a little service or app that automatically adds new music to a VLC playlist as I added new files to my collection. FileBeats seems like it might come in handy for watching my Music folders for new file additions.  However, FileBeats seemed like overkill for this exercise, so I decided to leave it for later.  **#MVP_FTW**

---
## Filter Section
**Grok**
```
filter {
  grok {
    match => { 
    "message" => "%{TIMESTAMP_ISO8601} %{WORD} %{WORD} %{NUMBER} ((- -)) %{WORD}=%{QS:description} %{WORD}=%{QS:hostname} %{WORD}=%{QS:source_ip} %{WORD}=%{QS:severity}"
    } 
  }
  ...
}
```
Ah, the main course of this exercise.  `grok` is absolutely amazing.  I'm pretty impressed with the number of built-in matchers it has.  Also, I like that it ignores data prior to the first matched expression.  I didn't even need to keep the custom matcher I made to match the `<14>1` part of the sample log message.  I was planning on using `(?<prefix><\d\d>\d)` to match the first part of the log.

I used `TIMESTAMP_ISO8601` instead of a `DATE` expression here so I wouldn't have to write out the date pattern.

Initially, I had all the matchers named, but for whatever reason, I couldn't get `remove_field` function to work in order to keep the output from getting cluttered with all the extra fields that didn't need to included. :( 

I would have prefered to have all of the matchers named, so that way it's easier to tell from the config what to each matcher is actually for.

`QS` (`QUOTEDSTRING`) was a double edged sword. It made it really easy to pull the values for `description`, `hostname`, `source_ip`, and `severity`.  But the values that `QS` matched included the extra `""` part of the value, which made the values REALLY hard to use in conditional statements and other plugins.  

---
**Mutate - gsub**
```
filter { 
  grok { ... }
  mutate {
    gsub => [
      "description", '"', "",
      "hostname", '"', "",
      "source_ip", '"', "",
      "severity", '"', ""
    ]
  }
```
I almost gave up using `QS` because the `""` were also getting matched and making it a pain to use the values in other plugins.  I'm glad I found `gsub` and was able to use it to cut out the extra `"` characters that were getting added by the `QS` matcher.

---
**Mutate with Condition**
```
filter { 
  grok { ... }
  mutate { ... }
 
  if [severity] == "1" { 
    # if there were more than just one severity that I needed to transform, I would have used an ENUM to map all of the input-output pairs.
    mutate {
      update => {
        "severity" => "High"
      }
    }
  }
}
```
Converting severity values of `1` to `High` was a major pain. :(

`if "1" in [severity] {...}` was the only way I could get a condition expresion to work with severity as an unedited QS value. Of course, this is bad because it would match any severity value with a "1" (i.e. "11", "123");

---
## Output Section
```
output {
  stdout { codec => rubydebug } 
  file {
    path => ".\data\cyderes-mwj\output.txt"
  }
  elasticsearch {
    hosts => ["http://localhost:9200"]
  }
}
```
Can we specify multiple codecs for a single input/output?  Just curious; I didn't get a chance to look into it. 

So far, I've been running Elasticsearch and Logstash in GitBash.  I'm still trying to figure out if it's necessary to use `\` for outputs on Windows, or if it's ok to use `/` in path values.  

Another fun caveat I discovered was that `Ctrl + C` only works to kill a running logstash instance when I use *logstash.bat*. That keyboard shortcut doesn't work when I try to use the *logstash -f \*.conf*.

Just for kicks, I configured Logstash to save logs to an OOTB `Elasticsearch` instance I was running running locally (also in a GitBash terminal).  Below is an example of the output I got when querying the document count for the ES index that logstash was using after I finished testing my filter: 
```
mjavier@Yukino MINGW64 /c/DevTools/LogStash/logstash-8.4.0/data/cyderes-mwj (main)
$ curl -X GET "localhost:9200/.ds-logs-generic-default-2022.08.29-000001/_count?pretty"
{
  "count" : 258,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}
```
---
## Final thoughts
I thought it was a fun exercise.  The `logstash.conf` very similar to the form configuration I had to do while working as a `Professional Services Engineer` for Firemon doing custom workflow form configuration.  Except for that, I was editing a `BPMN` (**glorified XML**) file and some JSON files for the form field layouts.  

I did have some minor annoyances working through the exercise.  I couldn't figure out a way to get `grok { match => { message => ... }}` to allow the matcher expressions to be configured on multiple lines instead of a single line which was frustrating.  I'm assuming that some of these grok-matchers can get pretty long.

Another thing, It was weird that logstash keeps looking for the .conf file based on the root installation directory, instead of the current working directory:
```
mjavier@Yukino MINGW64 /c/DevTools/LogStash/logstash-8.4.0/data/cyderes-mwj (main)
$ logstash.bat -f ./data/cyderes-mwj/mwj-logstash.conf --config.reload.automatic
```
It's strange that I have to specify the relative path from logstash_root (which I couldn't even find it explicity configured?) to my .conf file, even though I'm **IN** the directory that **HAS** my .conf file.  When I try to omit the `./data/folder/` part of the path, I get this message: 
```
mjavier@Yukino MINGW64 /c/DevTools/LogStash/logstash-8.4.0/data/cyderes-mwj (main)
$ logstash.bat -f mwj-logstash.conf --config.reload.automatic
"Using bundled JDK: C:\DevTools\LogStash\logstash-8.4.0\jdk\bin\java.exe"
Sending Logstash logs to C:/DevTools/LogStash/logstash-8.4.0/logs which is now configured via log4j2.properties
...
[2022-09-02T23:18:30,649][INFO ][logstash.config.source.local.configpathloader] No config files found in path {:path=>"C:/DevTools/LogStash/logstash-8.4.0/mwj-logstash.conf"}
[2022-09-02T23:18:30,653][ERROR][logstash.config.sourceloader] No configuration found in the configured sources.
[2022-09-02T23:18:30,889][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600, :ssl_enabled=>false}
[2022-09-02T23:18:33,699][INFO ][logstash.config.source.local.configpathloader] No config files found in path {:path=>"C:/DevTools/LogStash/logstash-8.4.0/mwj-logstash.conf"}
[2022-09-02T23:18:33,700][ERROR][logstash.config.sourceloader] No configuration found in the configured sources.

```

Other than these pretty minor quirks, I like Logstash.  I'm gonna stop rambling now.  I look forward to hearing your feedback on my logstash filter.