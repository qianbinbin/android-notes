# Activity
## Activity 生命周期
Android 中最常见的 Activity 回调方法有：onCreate、onStart、onResume、onPause、onStop、onDestroy 等。
从 Activity1 导航至 Activity2，先后调用：
- 1 onPause
- 2 onCreate、onStart、onResume
- 1 onSaveInstanceState、onStop

返回键跳转到 Activity1，先后调用：
- 2 onPause
- 1 onStart、onResume
- 2 onStop、onDestroy

值得注意的是，onSaveInstanceState 是只在有机会重新显示的情况下才会调用，例如屏幕旋转，导航至其它页面，按 HOME 键等。按返回键则不会，因为 Activity 已经被销毁，用户也不希望重新显示。onPause 会调用时，onSaveInstanceState 不一定会调用。onSaveInstancestate 适合保存控件数据，持久化数据应在 onPause 进行。

onRestoreInstanceState 并不与其成对使用，只在 Activity 被系统销毁后重新显示时才会调用。控件可以自动保存和恢复数据，例如 EditText 会保存当前输入的文本和选定状态。
如果不希望在屏幕旋转时销毁重建 Activity，可对 Activity 指定 configChanges 属性为 orientation，当 minSdkVersion 或 targetSdkVersion 大于 13 时，还需要指定 screenSize。

## Activity 的导航
### Task 任务
任务是完成一个特定工作时，用户交互的一系列相关 Activity，保存在栈中。同一个栈中的 Activity 可以来自不同的 App。

要更改 Task 的默认行为，可以修改清单文件中 &lt;activity&gt; 元素的相关属性：
- taskAffinity
- launchMode
- allowTaskReparenting
- clearTaskOnLaunch
- alwaysRetainTaskState
- finishOnTaskLaunch

也可以使用 Intent 的 flag：
- FLAG_ACTIVITY_NEW_TASK
- FLAG_ACTIVITY_CLEAR_TOP
- FLAG_ACTIVITY_SINGLE_TOP

### 定义 Activity 启动模式 launchMode
- standard，一律创建 Activity 的新实例。此为默认模式。
启动的 Activity 进入当前栈顶端，每个栈可以有此 Activity 的多个实例，每个 Activity 的实例也可存在于多个栈。
例如，栈中 Activity 实例有 ABCD（A 为栈底，D 为栈顶，下同），D 的 launchMode 为默认 standard，则启动 D 后栈变为 ABCDD。

- singleTop，栈顶仅有 Activity 的单个实例。
  + 如果请求 Activity 已经处于当前栈顶端，则不会创建其新的实例，因此不会调用 onCreate、onStart 方法，但会回调 onNewIntent 方法。
  + 否则仍会创建其实例。

 例如，栈中有 ABCD，D 的 launchMode 为 singleTop，则不会创建 D 的新实例，栈仍为 ABCD，且已有的 D 实例会回调 onNewIntent 方法。如果 ABCD 启动 B，则为 ABCDB，即使 B 的 launchMode 为 singleTop。

- singleTask，在所需任务栈中只能存在此 Activity 的单个实例。
不管目标 Activity 是在新的 Task 中还是在发动请求的 Task 中，返回键总是会将前一个 Activity 展示给用户。
首先系统会寻找请求 Activity 所需的任务栈。
  + 如果不存在所需任务栈，则创建任务栈，并创建 Activity 实例，压栈，目标 Activity 在栈底。
  + 如果存在所需任务栈：
    * 若此后台栈中存在目标 Activity 实例，则此整个栈回到前台，此时返回栈就会包括所有被调回前台的 Task 的 Activity，目标 Activity 实例处于当前栈顶，回调 onNewIntent 方法，以上元素（如果有）全部出栈；
    * 若栈中不存在所请求的 Activity，则创建此 Activity 实例，并压栈。

 例如，当前栈中有 1、2 两个 Activity，后台栈中有 X、Y 两个 Activity，Y 的 launchMode 声明为 singleTask，则打开 Y 后，返回栈中元素为 1、2、X、Y。
又例如，栈中有 AB 两个 Activity 实例，A 的 launchMode 设置为 singleTask，再次启动 A 后，栈中剩余元素为 A。

- singleInstance，Activity 仅以唯一的实例存在于一个单独的任务栈中。后续的请求均不会创建新的实例。

值得注意的是，在清单文件中为 Activity 定义的 launchMode 属性，都可以被打开此 Activity 的 Intent 的 flag 覆盖，也就是说 Intent 优先级高于 Activity 本身的 launchMode 属性。

### Intent 的 flag
- FLAG_ACTIVITY_NEW_TASK，[Google 官方文档](https://developer.android.com/guide/components/activities/tasks-and-back-stack.html)称与 singleTask 表现相同，实际上是不同的。
例如，栈中有 AB 两个 Activity，如果为 Intent 设置 FLAG_ACTIVITY_NEW_TASK，并启动 A，则栈变为 ABA，而不是像 singleTask 那样只剩下 A。
- FLAG_ACTIVITY_SINGLE_TOP，与 singleTop 表现相同。
- FLAG_ACTIVITY_CLEAR_TOP，如果目标 Activity 已在栈中，保持其它条件不变，假设在不设置此 flag 的情况下，
  + 目标 Activity 不会复用（standard 模式）：则在设置此 flag 后，目标 Activity 以上的实例全部出栈，本身也会被结束（finish），并重新创建实例。
  + 目标 Activity 会复用：则在设置此 flag 后，目标 Activity 以上的实例全部出栈，本身回调 onNewIntent 方法。

### taskAffinity 及其与 flag、launchMode 的结合使用
taskAffinity，指示 Activity 优先属于哪个task，默认为应用的包名，因此同一应用中的 Activity 默认属于同一个 task。
常与 Intent 的 flag 或者 Activity 的 launchMode 等结合使用。
- 与 singleTask 结合使用，参考上面 singleTask 部分。
- 与 FLAG_ACTIVITY_NEW_TASK 连用，与 singleTask 连用类似，但当目标 Activity 已经处于所需栈中、且 taskAffinity 相同时，不会导航到此 Activity，也不会回调其生命周期方法。
例如，栈中已有 AB 两个实例，A 的 taskAffinity 属性与此 task 相同，则 B 在启动 A 时，没有任何反应，B 不会被销毁，A 也不会被复用，栈元素保持不变。
- 与 FLAG_ACTIVITY_NEW_TASK 及 FLAG_ACTIVITY_CLEAR_TOP 连用，作用和 singleTask 类似，但 A 是否重用，要看 A 是否为默认 standard 模式，参考上面 FLAG_ACTIVITY_CLEAR_TOP 部分。
- 与 allowTaskReparenting 连用，若 Activity 的 allowTaskReparenting 属性为 true（默认为 false），打开此 Activity，然后进入后台，此时如有与此 Activity 相同的 TaskAffinity 的任务栈进入前台，则此 Activity 会调入此任务栈。
例如，从邮箱应用（假设为任务栈 S1）中点击一个链接，打开一个浏览器 Activity，压入 S1，显示一个网页，然后点击 HOME 键进入后台。如果其 allowTaskReparenting 属性为 true，此后用户点击浏览器应用，则显示的就是此 Activity 展示的网页；而在默认情况下，浏览器开启的是任务栈 S2，打开的是默认首页，这显然不符合用户期望。

## 隐式 Intent 及 IntentFilter
- Intent 显式调用：为 Intent 指定目标 Activity 或 Service 的 class，或者包含包名的类名。
- Intent 隐式调用：为 Intent 指定 action、category、data。例如，用户点击了一个链接，希望通过浏览器打开网页，使用隐式 Intent 就可以从支持的浏览器中的一个打开。Android 5.0 之后，Service 无法通过隐式 Intent 调用。
隐式 Intent 的过滤信息：
### action
Intent 中的 action 必须存在，且必须与过滤规则中的其中一个匹配。
### category
  - 如果 Intent 没有设置 category，则匹配成功。这是因为支持隐式 Intent 的 Activity 都必须声明`android.intent.category.DEFAULT`，而在启动 Activity 时默认会为 Intent 加入这个 category。
  - 如果 Intent 设置了 category，则必须与过滤规则中的其中一个匹配，才匹配成功。

 ### data
data 由两部分组成，URI 和 mimeType。
 - mimeType 为媒体类型，如 image/jpeg、video/\*、audio/mpeg4-generic 等。
 - URI 由 scheme、host、port、path/pathPattern/pathPrefix 组成。path 表示完整路径，pathPattern 也表示完整路径，但可以包含通配符，pathPrefix 表示路径前缀。

 data 的匹配规则和 action 类似，它要求 Intent 中必须包含 data 数据，且与过滤规则中的一个匹配。
intent-filter 中如果不指定 URI，则使用默认值 content 和 file，此时 Intent 中 URI 部分的 scheme 必须为 content 或 file 才能匹配。
为 Intent 指定 data 时，`setType(String type)`会将 data/URI 置为 null，`setData(Uri data)`也会将 type 置为 null，所以在需要同时指定两者时，必须使用`setDataAndType`方法。

 ### 判断是否存在与 Intent 匹配的 Activity
通过隐式方式启动 Activity 时，可以先判断是否有 Activity 能够匹配隐式 Intent，因为找不到 Activity 的话会抛出异常。判断方法有两种：
- `PackageManager`的`resolveActivity`方法
- `Intent`的`resolveActivity`方法

 如果上面两个方法找不到匹配的 Activity，则返回 null。需要注意的是，传递 flags 参数时，要使用`MATCH_DEFAULT_ONLY`这个标记位，因为支持隐式 Intent 的 Activity 必须包含 DEFAULT 的 category（见上面），如果不使用这个 flag，则返回的可能包括不支持隐式 Intent 的 Activity，startActivity 就会失败。

另外，在 intent-filter 中，MAIN 的 action 和 LAUNCHER 的 category 一起使用，表示这是一个入口 Activity，并且会出现在系统应用列表中，少了任何一个都没有实际意义。
