# 异步进程

为什么会学习到`child_process！`因为我最近接触到了一个关于使用ffmpeg的node库即[**node-fluent-ffmpeg**](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg)。

这个库是帮助更好操作ffmpeg API而生的。我很好奇它里面是如何实现的，因此就尝试去看了一下它的[源码](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg/blob/master/lib/processor.js#L4)。如果你有兴趣可以自己去读一下，对于学习node帮助还是很大的。

```js
  /**
   * Spawn an ffmpeg process
   *
   * The 'options' argument may contain the following keys:
   * - 'niceness': specify process niceness, ignored on Windows (default: 0)
   * - `cwd`: change working directory
   * - 'captureStdout': capture stdout and pass it to 'endCB' as its 2nd argument (default: false)
   * - 'stdoutLines': override command limit (default: use command limit)
   *
   * The 'processCB' callback, if present, is called as soon as the process is created and
   * receives a nodejs ChildProcess object.  It may not be called at all if an error happens
   * before spawning the process.
   *
   * The 'endCB' callback is called either when an error occurs or when the ffmpeg process finishes.
   *
   * @method FfmpegCommand#_spawnFfmpeg
   * @param {Array} args ffmpeg command line argument list
   * @param {Object} [options] spawn options (see above)
   * @param {Function} [processCB] callback called with process object and stdout/stderr ring buffers when process has been created
   * @param {Function} endCB callback called with error (if applicable) and stdout/stderr ring buffers when process finished
   * @private
   */
  proto._spawnFfmpeg = function(args, options, processCB, endCB) {
    // Enable omitting options
    if (typeof options === 'function') {
      endCB = processCB;
      processCB = options;
      options = {};
    }

    // Enable omitting processCB
    if (typeof endCB === 'undefined') {
      endCB = processCB;
      processCB = function() {};
    }

    var maxLines = 'stdoutLines' in options ? options.stdoutLines : this.options.stdoutLines;

    // Find ffmpeg
    this._getFfmpegPath(function(err, command) {
      if (err) {
        return endCB(err);
      } else if (!command || command.length === 0) {
        return endCB(new Error('Cannot find ffmpeg'));
      }

      // Apply niceness
      if (options.niceness && options.niceness !== 0 && !utils.isWindows) {
        args.unshift('-n', options.niceness, command);
        command = 'nice';
      }

      var stdoutRing = utils.linesRing(maxLines);
      var stdoutClosed = false;

      var stderrRing = utils.linesRing(maxLines);
      var stderrClosed = false;

      // Spawn process
      var ffmpegProc = spawn(command, args, options);

      if (ffmpegProc.stderr) {
        ffmpegProc.stderr.setEncoding('utf8');
      }

      ffmpegProc.on('error', function(err) {
        endCB(err);
      });

      // Ensure we wait for captured streams to end before calling endCB
      var exitError = null;
      function handleExit(err) {
        if (err) {
          exitError = err;
        }

        if (processExited && (stdoutClosed || !options.captureStdout) && stderrClosed) {
          endCB(exitError, stdoutRing, stderrRing);
        }
      }

      // Handle process exit
      var processExited = false;
      ffmpegProc.on('exit', function(code, signal) {
        processExited = true;

        if (signal) {
          handleExit(new Error('ffmpeg was killed with signal ' + signal));
        } else if (code) {
          handleExit(new Error('ffmpeg exited with code ' + code));
        } else {
          handleExit();
        }
      });

      // Capture stdout if specified
      if (options.captureStdout) {
        ffmpegProc.stdout.on('data', function(data) {
          stdoutRing.append(data);
        });

        ffmpegProc.stdout.on('close', function() {
          stdoutRing.close();
          stdoutClosed = true;
          handleExit();
        });
      }

      // Capture stderr if specified
      ffmpegProc.stderr.on('data', function(data) {
        stderrRing.append(data);
      });

      ffmpegProc.stderr.on('close', function() {
        stderrRing.close();
        stderrClosed = true;
        handleExit();
      });

      // Call process callback
      processCB(ffmpegProc, stdoutRing, stderrRing);
    });
  };
```

解读一下

```js
  /**
   * 产生一个ffmpeg进程
   *
   * 'options' argument may contain the following keys:
   * - 'niceness': specify process niceness, ignored on Windows (default: 0)
   * - `cwd`: change working directory
   * - 'captureStdout': capture stdout and pass it to 'endCB' as its 2nd argument (default: false)
   * - 'stdoutLines': override command limit (default: use command limit)
   *
   * The 'processCB' callback, if present, is called as soon as the process is created and
   * receives a nodejs ChildProcess object.  It may not be called at all if an error happens
   * before spawning the process.
   *
   * The 'endCB' callback is called either when an error occurs or when the ffmpeg process finishes.
   *
   * @method FfmpegCommand#_spawnFfmpeg
   * @param {Array} ffmpeg命令数组，这个看一下ffmpeg即可
   * @param {Object} [options] spawn options (see above)
   * @param {Function} [processCB] callback called with process object and stdout/stderr ring buffers when process has been created
   * @param {Function} endCB callback called with error (if applicable) and stdout/stderr ring buffers when process finished
   * @private
   */
  proto._spawnFfmpeg = function(args, options, processCB, endCB) {
    // Enable omitting options
    if (typeof options === 'function') {
      endCB = processCB;
      processCB = options;
      options = {};
    }

    // Enable omitting processCB
    if (typeof endCB === 'undefined') {
      endCB = processCB;
      processCB = function() {};
    }

    var maxLines = 'stdoutLines' in options ? options.stdoutLines : this.options.stdoutLines;

    // Find ffmpeg
    this._getFfmpegPath(function(err, command) {
      if (err) {
        return endCB(err);
      } else if (!command || command.length === 0) {
        return endCB(new Error('Cannot find ffmpeg'));
      }

      // Apply niceness
      if (options.niceness && options.niceness !== 0 && !utils.isWindows) {
        args.unshift('-n', options.niceness, command);
        command = 'nice';
      }

      var stdoutRing = utils.linesRing(maxLines);
      var stdoutClosed = false;

      var stderrRing = utils.linesRing(maxLines);
      var stderrClosed = false;

      // Spawn process
      var ffmpegProc = spawn(command, args, options);

      if (ffmpegProc.stderr) {
        ffmpegProc.stderr.setEncoding('utf8');
      }

      // 知识点 child_pro
      ffmpegProc.on('error', function(err) {
        endCB(err);
      });

      // Ensure we wait for captured streams to end before calling endCB
      var exitError = null;
      function handleExit(err) {
        if (err) {
          exitError = err;
        }

        if (processExited && (stdoutClosed || !options.captureStdout) && stderrClosed) {
          endCB(exitError, stdoutRing, stderrRing);
        }
      }

      // Handle process exit
      var processExited = false;
      ffmpegProc.on('exit', function(code, signal) {
        processExited = true;

        if (signal) {
          handleExit(new Error('ffmpeg was killed with signal ' + signal));
        } else if (code) {
          handleExit(new Error('ffmpeg exited with code ' + code));
        } else {
          handleExit();
        }
      });

      // Capture stdout if specified
      if (options.captureStdout) {
        ffmpegProc.stdout.on('data', function(data) {
          stdoutRing.append(data);
        });

        ffmpegProc.stdout.on('close', function() {
          stdoutRing.close();
          stdoutClosed = true;
          handleExit();
        });
      }

      // Capture stderr if specified
      ffmpegProc.stderr.on('data', function(data) {
        stderrRing.append(data);
      });

      ffmpegProc.stderr.on('close', function() {
        stderrRing.close();
        stderrClosed = true;
        handleExit();
      });

      // Call process callback
      processCB(ffmpegProc, stdoutRing, stderrRing);
    });
  };
```



