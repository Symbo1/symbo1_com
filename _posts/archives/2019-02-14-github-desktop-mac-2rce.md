---
layout: post
title: Github Desktop RCEx2 for Mac latest Version
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - CongRong</p>

截至今日最新版的两处RCE，第一个重复@evi1m0/第二个内部排查已知，复现蛮简单的，直接show your code.

## 0x01

app/src/ui/diff/binary-file.tsx:

{% highlight javascript %}
export class BinaryFile extends React.Component<IBinaryFileProps, {}> {
  private open = () => {
    const fullPath = Path.join(this.props.repository.path, this.props.path)
    openFile(fullPath, this.props.dispatcher)
  }

  public render() {
    return (
      <div className="panel binary" id="diff">
        <div className="image-header">This binary file has changed.</div>
        <div className="image-header">
          <LinkButton onClick={this.open}>
            Open file in external program.
          </LinkButton>
        </div>
      </div>
    )
  }
}
{% endhighlight %}

Follow up `openFile`

app/src/ui/lib/open-file.ts:

{% highlight javascript %}
const result = await shell.openExternal(`file://${fullPath}`)
{% endhighlight %}

## 0x02

app/src/ui/lib/context-menu.ts:

{% highlight javascript %}
const RestrictedFileExtensions = ['.cmd', '.exe', '.bat', '.sh']
export const CopyFilePathLabel = __DARWIN__
  ? 'Copy File Path'
  : 'Copy file path'

export const DefaultEditorLabel = __DARWIN__
  ? 'Open in External Editor'
  : 'Open in external editor'

export const RevealInFileManagerLabel = __DARWIN__
  ? 'Reveal in Finder'
  : __WIN32__
  ? 'Show in Explorer'
  : 'Show in your File Manager'

export const TrashNameLabel = __DARWIN__ ? 'Trash' : 'Recycle Bin'

export const OpenWithDefaultProgramLabel = __DARWIN__
  ? 'Open with Default Program'
  : 'Open with default program'

export function isSafeFileExtension(extension: string): boolean {
  if (__WIN32__) {
    return RestrictedFileExtensions.indexOf(extension.toLowerCase()) === -1
  }
  return true
}
{% endhighlight %}

`context-menu` restricted these files extensions:

{% highlight javascript %}
const RestrictedFileExtensions = ['.cmd', '.exe', '.bat', '.sh']
{% endhighlight %}

But macos is a Unix OS, and many binary files can be run directly.

Follow up and search `OpenWithDefaultProgramLabel`:

app/src/ui/changes/changes-list.tsx:

{% highlight javascript %}
24    CopyFilePathLabel,
25    RevealInFileManagerLabel,
26:   OpenWithDefaultProgramLabel,
27  } from '../lib/context-menu'
28  import { CommitMessage } from './commit-message'
..
374        },
375        {
376:         label: OpenWithDefaultProgramLabel,
377          action: () => this.props.onOpenItem(path),
378          enabled: isSafeExtension && status.kind !== AppFileStatusKind.Deleted,
{% endhighlight %}

app/src/ui/history/file-list.tsx:

{% highlight javascript %}
13    DefaultEditorLabel,
14    RevealInFileManagerLabel,
15:   OpenWithDefaultProgramLabel,
16  } from '../lib/context-menu'
17  import { List } from '../lib/list'
 ..
150        },
151        {
152:         label: OpenWithDefaultProgramLabel,
153          action: () => this.props.onOpenItem(filePath),
154          enabled: isSafeExtension && fileExistsOnDisk,
{% endhighlight %}

app/src/ui/merge-conflicts/merge-conflicts-dialog.tsx:

{% highlight javascript %}
27  import { IMenuItem } from '../../lib/menu-item'
28  import {
29:   OpenWithDefaultProgramLabel,
30    RevealInFileManagerLabel,
31  } from '../lib/context-menu'
..
291        const items: IMenuItem[] = [
292          {
293:           label: OpenWithDefaultProgramLabel,
294            action: () => openFile(absoluteFilePath, dispatcher),
295          },
{% endhighlight %}

These calls can be used under macos, not subject to file extensions.