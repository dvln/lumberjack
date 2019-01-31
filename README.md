# lumberjack  [![GoDoc](https://godoc.org/gopkg.in/natefinch/lumberjack.v2?status.png)](https://godoc.org/gopkg.in/natefinch/lumberjack.v2) [![Build Status](https://travis-ci.org/natefinch/lumberjack.svg?branch=v2.0)](https://travis-ci.org/natefinch/lumberjack) [![Build status](https://ci.appveyor.com/api/projects/status/00gchpxtg4gkrt5d)](https://ci.appveyor.com/project/natefinch/lumberjack) [![Coverage Status](https://coveralls.io/repos/natefinch/lumberjack/badge.svg?branch=v2.0)](https://coveralls.io/r/natefinch/lumberjack?branch=v2.0)

### Lumberjack is a Go package for writing logs to rolling files.

Package lumberjack provides a rolling logger.

Note: Local modifications to lumberjack are accessed from the master branch in
this git repo if you want to use this vendored copy:

    import "github.com/dvln/lumberjack"

The primary difference of this version is that the name of the log files are
a little differently structured and it has a feature to do daily rotation of
the log file as well as allow a simple date extension on the log file without
time... and can handle multiple log files of the same name on the same date
(such as if the file exceeds the desired max size of the file).  There is
more detail on this below (see Note1, Note2 and Note3 below).

If you want to use the original version of this tool (from Nate Finch on github) then
read the below branch and import information as that is from the original package which
is using the v2.0 branch.

Note that this is v2.0 of lumberjack, and should be imported using gopkg.in
thusly:

    import "gopkg.in/natefinch/lumberjack.v2"

The package name remains simply lumberjack, and the code resides at
https://github.com/natefinch/lumberjack under the v2.0 branch.

Lumberjack is intended to be one part of a logging infrastructure.
It is not an all-in-one solution, but instead is a pluggable
component at the bottom of the logging stack that simply controls the files
to which logs are written.

Lumberjack plays well with any logging package that can write to an
io.Writer, including the standard library's log package.

Lumberjack assumes that only one process is writing to the output files.
Using the same lumberjack configuration from multiple processes on the same
machine will result in improper behavior.


**Example**

To use lumberjack with the standard library's log package, just pass it into the SetOutput function when your application starts.

Code:

```go
log.SetOutput(&lumberjack.Logger{
    Filename:    "/var/log/myapp/foo.log",
    MaxSize:     500, // megabytes
    MaxBackups:  3,
    DailyRotate: true, // disabled by default
    LocalTime:   true, // disabled by default
    MaxAge:      28, //days
    Compress:    true, // disabled by default
})
```

Note1: DailyRotate is an local addition.  It is used to rotate log files at midnight
       local time so it kind of follows Apache and other logs file formats with
       nightly rotation to make it easy to group log files by a given days activity.
       It should be noted that the default log file naming is different than in the
       central copy of this package, see next note.

Note2: For dvln purposes the default log format is like tool.log[.2019-01-15[--<#>][.gz]]
       so as to make the date quite clear.  If the MaxSize is exceeded within that day
       then the "--<#>" will kick in to keep the log file names unique within that day
       (the central copy of this pkg will just overwrite the prior archived log file
       if the size is reached on the same day if using a simple date extension, but
       that is only because it defaults to a hard coded date/time, down to milliseconds,
       and doesn't really have to worry about naming collisions due to that fact, so
       the "--#" optional extension is unique to this copy of the package as well).
       This copy of the package offer some API's to tweak the default date/time format
       more dynamically... but the default is to just identify the date without time
       details.  Note that the .gz (gzip) extension is only seen when compression is
       turned on.  The "former" central lumberjack log name format, which you might
       see in the below docs, was "tool[-2019-01-15T10-15-33.021].log[.gz]"... that
       exact format is no longer available but tool.log[.2019-01-15T10-15-33.021[.gz]]
       is available if one wants more detailed date/time still (see new API's to set
       the backup time format).  Anyhow, the point is that this files contents haven't
       been updated beyond these notes so you might want to keep that in mind.

Note3: The v2 branch in nate finch's central copy of this pkg is the latest public code
       to use at the time of this writing.  The v2 branch in this repo is "pristine" and
       tracks that (although it will grow out of date, I tend to update it if I want
       to merge in some features... most recently, compression).  So, if you want to
       bring in new changes from the standard repo put it in v2 cleanly and then
       merge that into the locally hacked version on master.  It should be noted
       that if the changes are big you might have a fair chunk of work in front of
       you since this has been changed a fair amount.

## type Logger
``` go
type Logger struct {
    // Filename is the file to write logs to.  Backup log files will be retained
    // in the same directory.  It uses <processname>-lumberjack.log in
    // os.TempDir() if empty.
    Filename string `json:"filename" yaml:"filename"`

    // MaxSize is the maximum size in megabytes of the log file before it gets
    // rotated. It defaults to 100 megabytes.
    MaxSize int `json:"maxsize" yaml:"maxsize"`

    // MaxAge is the maximum number of days to retain old log files based on the
    // timestamp encoded in their filename.  Note that a day is defined as 24
    // hours and may not exactly correspond to calendar days due to daylight
    // savings, leap seconds, etc. The default is not to remove old log files
    // based on age.
    MaxAge int `json:"maxage" yaml:"maxage"`

    // MaxBackups is the maximum number of old log files to retain.  The default
    // is to retain all old log files (though MaxAge may still cause them to get
    // deleted.)
    MaxBackups int `json:"maxbackups" yaml:"maxbackups"`

    // LocalTime determines if the time used for formatting the timestamps in
    // backup files is the computer's local time.  The default is to use UTC
    // time.
    LocalTime bool `json:"localtime" yaml:"localtime"`

    // Compress determines if the rotated log files should be compressed
    // using gzip. The default is not to perform compression.
    Compress bool `json:"compress" yaml:"compress"`
    // contains filtered or unexported fields
}
```
Logger is an io.WriteCloser that writes to the specified filename.

Logger opens or creates the logfile on first Write.  If the file exists and
is less than MaxSize megabytes, lumberjack will open and append to that file.
If the file exists and its size is >= MaxSize megabytes, the file is renamed
by putting the current time in a timestamp in the name immediately before the
file's extension (or the end of the filename if there's no extension). A new
log file is then created using original filename.

Whenever a write would cause the current log file exceed MaxSize megabytes,
the current file is closed, renamed, and a new log file created with the
original name. Thus, the filename you give Logger is always the "current" log
file.

Backups use the log file name given to Logger, in the form `name-timestamp.ext`
where name is the filename without the extension, timestamp is the time at which
the log was rotated formatted with the time.Time format of
`2006-01-02T15-04-05.000` and the extension is the original extension.  For
example, if your Logger.Filename is `/var/log/foo/server.log`, a backup created
at 6:30pm on Nov 11 2016 would use the filename
`/var/log/foo/server-2016-11-04T18-30-00.000.log`

### Cleaning Up Old Log Files
Whenever a new logfile gets created, old log files may be deleted.  The most
recent files according to the encoded timestamp will be retained, up to a
number equal to MaxBackups (or all of them if MaxBackups is 0).  Any files
with an encoded timestamp older than MaxAge days are deleted, regardless of
MaxBackups.  Note that the time encoded in the timestamp is the rotation
time, which may differ from the last time that file was written to.

If MaxBackups and MaxAge are both 0, no old log files will be deleted.











### func (\*Logger) Close
``` go
func (l *Logger) Close() error
```
Close implements io.Closer, and closes the current logfile.



### func (\*Logger) Rotate
``` go
func (l *Logger) Rotate() error
```
Rotate causes Logger to close the existing log file and immediately create a
new one.  This is a helper function for applications that want to initiate
rotations outside of the normal rotation rules, such as in response to
SIGHUP.  After rotating, this initiates a cleanup of old log files according
to the normal rules.

**Example**

Example of how to rotate in response to SIGHUP.

Code:

```go
l := &lumberjack.Logger{}
log.SetOutput(l)
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGHUP)

go func() {
    for {
        <-c
        l.Rotate()
    }
}()
```

### func (\*Logger) Write
``` go
func (l *Logger) Write(p []byte) (n int, err error)
```
Write implements io.Writer.  If a write would cause the log file to be larger
than MaxSize, the file is closed, renamed to include a timestamp of the
current time, and a new log file is created using the original log file name.
If the length of the write is greater than MaxSize, an error is returned.









- - -
Generated by [godoc2md](http://godoc.org/github.com/davecheney/godoc2md)
