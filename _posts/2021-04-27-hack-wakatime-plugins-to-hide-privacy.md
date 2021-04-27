---
layout: post
title: How to hack wakatime plugins to hide privacy?
categories: [productivity]
tags: [wakatime]
---

I use [`WakaTime`] to metrics on my time when using computers since 2016.
But I don't want to expose any of my privacy. So I hack it.

#### The Location of the Wakatime Command Line Interface

For most wakatime plugins, they will invoke the wakatime cli in `${PYTHONPATH}`,
such as [`wakatime-mode` for Emacs] or [wakatime plugin for Bash].

But for others, they will invoke the cli which in the same directory as those
plugins, such as [`vim-wakatime` for Vim] or [`sublime-wakatime` for Sublime].

#### How to Hide Privacy?

##### Hide File Names

Update [`Heartbeat.sanitize(..)` in "wakatime/heartbeat.py"]:

```diff
  if self._should_obfuscate_filename():
      self._sanitize_metadata(keys=self._sensitive_when_hiding_filename)
      if self._should_obfuscate_branch(default=True):
          self._sanitize_metadata(keys=self._sensitive_when_hiding_branch)
      extension = u(os.path.splitext(self.entity)[1])
-     self.entity = u('HIDDEN{0}').format(extension)
+     self.entity = u('any-file-name-as-you-like')
  elif self.should_obfuscate_project():
      self._sanitize_metadata(keys=self._sensitive_when_hiding_filename)
      if self._should_obfuscate_branch(default=True):
          self._sanitize_metadata(keys=self._sensitive_when_hiding_branch)
  elif self._should_obfuscate_branch():
      self._sanitize_metadata(keys=self._sensitive_when_hiding_branch)
```

##### Hide Text Editors

- Select a faked user agent.

  For example, [the user agent for Visual Studio Code].

- Update wakatime plugins for your text editors.

  For examples:

  - For Emacs, update [`wakatime-user-agent` in "wakatime-mode/wakatime-mode.el"]:

    ```diff
    - (defconst wakatime-user-agent "emacs-wakatime")
    + (defconst wakatime-user-agent "any-user-agent-as-you-like")
    ```

  - For Vim, update [`s:SendHeartbeats(..)` in "vim-wakatime/plugin/wakatime.vim"]:

    ```diff
    - let cmd = cmd + ['--plugin', printf('vim/%s vim-wakatime/%s', s:n2s(v:version), s:VERSION)]
    + let cmd = cmd + ['--plugin', 'any-user-agent-as-you-like']
    ```

  - For Sublime, update [`SendHeartbeatsThread.send_heartbeats(..)` in "WakaTime/WakaTime.py"]:

    ```diff
    - ua = 'sublime/%d sublime-wakatime/%s' % (ST_VERSION, __version__)
    + ua = 'any-user-agent-as-you-like'
    ```

    Notice that there are more than one `ua = ..`, you should update all of them.

##### Hide Operating Systems

- Select a faked platform.

  ```sh
  python -c "import platform; print(platform.platform());"
  ```

- **(Optional)** Select a faked user agent.

  For example, [the user agent for Visual Studio Code].

  Do this will hide the name of text editors, same effect as the previous section.

- Update [`get_user_agent(..)` in "wakatime/utils.py"]:

  ```diff
    def get_user_agent(plugin=None):
  +     CUSTOMIZED_PLUGIN = 'any-plugin-as-you-like'
  +     CUSTOMIZED_PLATFORM = 'any-platform-as-you-like'
        ver = sys.version_info
        python_version = '%d.%d.%d.%s.%d' % (ver[0], ver[1], ver[2], ver[3], ver[4])
        user_agent = u('wakatime/{ver} ({platform}) Python{py_ver}').format(
            ver=u(__version__),
  -         platform=u(platform.platform()),
  +         platform=u(CUSTOMIZED_PLATFORM),
            py_ver=python_version,
        )
  -     if plugin:
  -         user_agent = u('{user_agent} {plugin}').format(
  -             user_agent=user_agent,
  -             plugin=u(plugin),
  -         )
  -     else:
  -         user_agent = u('{user_agent} Unknown/0').format(
  -             user_agent=user_agent,
  -         )
  +     user_agent = u('{user_agent} {plugin}').format(
  +         user_agent=user_agent,
  +         plugin=u(CUSTOMIZED_PLUGIN),
  +     )
        return user_agent
  ```

[`WakaTime`]: https://wakatime.com/
[`wakatime-mode` for Emacs]: https://github.com/wakatime/wakatime-mode
[`vim-wakatime` for Vim]: https://github.com/wakatime/vim-wakatime
[`sublime-wakatime` for Sublime]: https://github.com/wakatime/sublime-wakatime
[wakatime plugin for Bash]: https://wakatime.com/terminal
[`Heartbeat.sanitize(..)` in "wakatime/heartbeat.py"]: https://github.com/wakatime/wakatime/blob/13.1.0/wakatime/heartbeat.py#L155-L166
[the user agent for Visual Studio Code]: https://github.com/wakatime/vscode-wakatime/blob/5.0.1/src/wakatime.ts#L441-L442
[`wakatime-user-agent` in "wakatime-mode/wakatime-mode.el"]: https://github.com/wakatime/wakatime-mode/blob/7626678315918bdbb81ede68149f20a7d97a928f/wakatime-mode.el#L37
[`s:SendHeartbeats(..)` in "vim-wakatime/plugin/wakatime.vim"]: https://github.com/wakatime/vim-wakatime/blob/8.0.0/plugin/wakatime.vim#L345
[`SendHeartbeatsThread.send_heartbeats(..)` in "WakaTime/WakaTime.py"]: https://github.com/wakatime/sublime-wakatime/blob/10.0.1/WakaTime.py#L695-L755
[`get_user_agent(..)` in "wakatime/utils.py"]: https://github.com/wakatime/wakatime/blob/13.1.0/wakatime/utils.py#L57-L74
