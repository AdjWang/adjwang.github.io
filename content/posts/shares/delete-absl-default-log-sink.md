---
title: "\"Delete\" abseil's default log sink"
summary: ""
date: 2024-09-15T21:06:00+08:00
table_of_contents: true
---

# "Delete" abseil's default log sink

## Abseil's logging library

I like google's code syntax of logging, it's more clean than spdlog. The log library is available from [glog](https://github.com/google/glog) or [abseil](https://github.com/abseil/abseil-cpp). Abseil also contains a bunch of utilities and remains quite lightweight. (Yes, I'm looking at you, boost). It's so great to have all of them by adding only one dependency.

This is the first time I'm working with abseil's logging module with vscode and got some trubble.

The abseil version used in this post is `tag: 20240722.0`.

## Problem in vscode

Create a new project, grab abseil as a submodule, make a deadly simple main function and log something in vscode's debug console(press Ctrl+Shift+Y to show)... Everything works smoothly except -- the log prints twice!

## The cause

Code from `abseil\absl\log\internal\log_sink_set.cc` shows what happens. When compiling on windows, abseil actually adds 2 default log sinks.

```cpp
// abseil\absl\log\internal\log_sink_set.cc
class GlobalLogSinkSet final {
 public:
  GlobalLogSinkSet() {
#if defined(__myriad2__) || defined(__Fuchsia__)
    // myriad2 and Fuchsia do not log to stderr by default.
#else
    static absl::NoDestructor<StderrLogSink> stderr_log_sink;
    AddLogSink(stderr_log_sink.get());
#endif
...
#if defined(_WIN32)
    static absl::NoDestructor<WindowsDebuggerLogSink> debugger_log_sink;
    AddLogSink(debugger_log_sink.get());
#endif  // !defined(_WIN32)
  }
...
```

It is clear that both `stderr_log_sink` and `debugger_log_sink` are added in the global log sink set constructor. After that, when some log messages coming in, they are dispatched to all of the sinks. In console, logs only from the stderr sink are captured, while in [DebugView](https://learn.microsoft.com/en-us/sysinternals/downloads/debugview), logs only from the windows debugger sink are captured, they works fine. However in vscode debug console, logs from both sinks are captured!

## Log to only one sink

It would be nice if abseil provides some interfaces like `RemoveLogSink(DefaultSinkType sink)`. However, it only provides `RemoveLogSink(absl::LogSink* sink)` in `abseil\absl\log\log_sink_registry.h`. The parameter is a pointer to sink. All the sinks you passed here are some one must have been added through another explicitly invoked interface `AddLogSink(absl::LogSink* sink)` in the user code. This design prevents user from removing any default sinks.

I didn't find any way to make vscode's debug console to only capture 1 log source. Modify abseil source code maybe a possible solution but not graceful.

Despite unable to remove a default sink, the library provides another interesting interface `LogMessage::ToSinkOnly(absl::LogSink* sink)`. There's also a brief introduction in [abseil's document](https://abseil.io/docs/cpp/guides/logging#LogSink).

```cpp
// abseil\absl\log\internal\log_message.cc
LogMessage& LogMessage::ToSinkOnly(absl::LogSink* sink) {
  ABSL_INTERNAL_CHECK(sink, "null LogSink*");
  data_->extra_sinks.clear();
  data_->extra_sinks.push_back(sink);
  data_->extra_sinks_only = true;
  return *this;
}
```

After passing an existing sink is passed in, method `SendToLog` is called, then the `extra_sinks_only` flag would get the sink to override all existing ones.

```cpp
// abseil\absl\log\internal\log_message.cc
void LogMessage::SendToLog() {
  ...
  // Also log to all registered sinks, even if OnlyLogToStderr() is set.
  log_internal::LogToSinks(data_->entry, absl::MakeSpan(data_->extra_sinks),
                           data_->extra_sinks_only);
  ...
}

// abseil\absl\log\internal\log_sink_set.h
// This function may log to two sets of sinks:
//
// * If `extra_sinks_only` is true, it will dispatch only to `extra_sinks`.
//   `LogMessage::ToSinkAlso` and `LogMessage::ToSinkOnly` are used to attach
//    extra sinks to the entry.
// * Otherwise it will also log to the global sinks set. This set is managed
//   by `absl::AddLogSink` and `absl::RemoveLogSink`.
void LogToSinks(const absl::LogEntry& entry,
                absl::Span<absl::LogSink*> extra_sinks, bool extra_sinks_only);
```

Therefore if the code always logging with an outside sink, logs would be printed through the new sink rather than default ones. Default sinks are not deleted here, they are just leaved away.

## Implementation

Abseil exports log interfaces by macros, like `LOG`, `PLOG`, or `DLOG`. To inject the `ToSinkOnly` into the middle, we first have to implement a custom sink object, then reimplement the log macros.

### Custom sinks

It's quite easy to implement a stderr sink and a windows debugger sink, since abseil already have them.

Just copy them from `abseil\absl\log\internal\log_sink_set.cc`, and add a static member as a singleton, a log sink selection function that gets the proper sink by macro settings. Now create a new header names `logging.h` in the project.

```cpp
// logging.h
#pragma once
#include <absl/log/globals.h>
#include <absl/log/internal/globals.h>
#include <absl/log/log_sink.h>

namespace proj_ns {
namespace logging {
class StderrLogSink final : public absl::LogSink {
public:
  static absl::LogSink *get() { return &kLogSink; }
  ~StderrLogSink() override = default;

  void Send(const absl::LogEntry &entry) override {
    if (entry.log_severity() < absl::StderrThreshold()) {
      return;
    }
    if (!entry.stacktrace().empty()) {
      absl::log_internal::WriteToStderr(entry.stacktrace(),
                                        entry.log_severity());
    } else {
      absl::log_internal::WriteToStderr(
          entry.text_message_with_prefix_and_newline(), entry.log_severity());
    }
  }

private:
  static StderrLogSink kLogSink;
};

class WindowsDebuggerLogSink final : public absl::LogSink {
public:
  static absl::LogSink *get() { return &kLogSink; }
  ~WindowsDebuggerLogSink() override = default;

  void Send(const absl::LogEntry &entry) override {
    if (entry.log_severity() < absl::StderrThreshold() &&
        absl::log_internal::IsInitialized()) {
      return;
    }
    ::OutputDebugStringA(entry.text_message_with_prefix_and_newline_c_str());
  }

private:
  static WindowsDebuggerLogSink kLogSink;
};

inline absl::LogSink *GetLogSink() {
#if defined LOG_SINK_DBGVIEW
  return WindowsDebuggerLogSink::get();
#elif defined LOG_SINK_STD
  return StderrLogSink::get();
#else
#error "Invalid log sink."
#endif
}
} // namespace logging
} // namespace proj_ns
```

Also remember to create a source file to define the static members.

```cpp
// logging.cc
#include "logging.h"
namespace proj_ns {
namespace logging {
#if defined LOG_SINK_DBGVIEW
WindowsDebuggerLogSink WindowsDebuggerLogSink::kLogSink;
#elif defined LOG_SINK_STD
StderrLogSink StderrLogSink::kLogSink;
#endif
} // namespace logging
} // namespace proj_ns
```

### Log macros

Here use `LOG` as an example to show how to reimplement macros.

Log macros are defined in separate headers as follow. They are listed in dependency order here.

1. `LOG` in `abseil\absl\log\log.h`

2. `ABSL_LOG_INTERNAL_LOG_IMPL(_##severity)` in `abseil\absl\log\internal\log_impl.h`

3. `ABSL_LOGGING_INTERNAL_LOG##severity` in `abseil\absl\log\internal\strip.h`

4. `ABSL_LOG_INTERNAL_CONDITION##severity` (used by the `ABSL_LOG_INTERNAL_LOG_IMPL` macro) in `abseil\absl\log\internal\conditions.h`.

Where the `ABSL_LOG_INTERNAL_LOG_IMPL(_##severity)` is the place to put the `ToSinkOnly`. And copy essential macros from headers above.

```cpp
// logging.h
...
// ABSL_LOG()
#define EXABSL_LOG_INTERNAL_LOG_IMPL(severity)           \
  ABSL_LOG_INTERNAL_CONDITION##severity(STATELESS, true) \
      ABSL_LOGGING_INTERNAL_LOG##severity                \
          .ToSinkOnly(proj_ns::logging::GetLogSink())    \
          .InternalStream()

// LOG()
//
// `LOG` takes a single argument which is a severity level.  Data streamed in
// comprise the logged message.
// Example:
//
//   LOG(INFO) << "Found " << num_cookies << " cookies";
#define LOG(severity) EXABSL_LOG_INTERNAL_LOG_IMPL(_##severity)
```

An `EX` prefix is added to the macro, which means "extended", to avoid name conflictions with macros that already defined in `log_impl.h`.

Rest macros like `PLOG`, `DLOG` can be added by the same way.

## Thread safety

Logging through extra sinks still goes to `GlobalLogSinkSet::LogToSinks`. However, extra sinks are not guarded by the internal mutex of `GlobalLogSinkSet`.

```cpp
// abseil\absl\log\internal\log_sink_set.cc
class GlobalLogSinkSet final {
  ...
  void LogToSinks(const absl::LogEntry& entry,
                  absl::Span<absl::LogSink*> extra_sinks, bool extra_sinks_only)
      ABSL_LOCKS_EXCLUDED(guard_) {
    SendToSinks(entry, extra_sinks);

    if (!extra_sinks_only) {
      if (ThreadIsLoggingToLogSink()) {
        ...
      } else {
        absl::ReaderMutexLock global_sinks_lock(&guard_);
        ThreadIsLoggingStatus() = true;
        ...
        SendToSinks(entry, absl::MakeSpan(sinks_));
      }
    }
  }
  ...
```

It requires additonal efforts to make the logging thread safe. It could be simply adding a mutex in the custom sink class or use some more complicated way like abseil's. I'm not going to touch it here.

## Finally

Now replace `#include <absl/log/log.h>` with the new log header file, logs are only sent to 1 sink. That's it! The behaviour is similar with that default sinks are "deleted".
