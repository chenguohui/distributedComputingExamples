# Hadoop Examples

[Apache](http://hadoop.apache.org/) [Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop) is a framework for performing large-scale distributed computations in a cluster. Different from [MPI](https://en.wikipedia.org/wiki/Message_Passing_Interface) (which we discussed [here](http://github.com/thomasWeise/distributedComputingExamples/tree/master/mpi/)), it is based on Java technologies. It may thus be slower than MPI, but can reap the full benefit of the rich libraries and programming environment of the Java ecosystem (just think about all the things we already did in this course). One of the most well-known ways to use Hadoop is to perform computations following the [MapReduce](https://en.wikipedia.org/wiki/MapReduce) pattern (which is a tiny little bit similar to scatter/gather/reduce in MPI).

Let us now shortly compare use cases of MPI versus MapReduce with Hadoop. MPI is the technology of choice if

- communication is expensive and the bottleneck of our application,
- frequent communication is required between processes solving related sub-problems,
- the available hardware is homogenous (and we can use an MPI implementation optimized for it),
- processes need to be organized in groups or topological structures to make efficient use of collective communication to achieve high performance,
- the size of data that needs to be transmitted is smaller in comparison to runtime of computations, and when
- we do not need to worry much about exchanging data with a heterogeneous distributed application environment.

Hadoop, on the other hand, covers use cases where

- communication is not the bottleneck, because computation takes much longer than communication (think Machine Learning), when
- the environment is heterogeneous,
- processes do not need to be organized in a special way and
- the division of tasks into sub-problems can be done efficiently by just slicing the input data into equal-sized pieces, where
- sub-problems have batch job character, where
- data is unstructured (e.g., text) and potentially huge (eating away the advantages of MPI-style communication), or where
- data comes from and results must be pushed back to other applications in the environment, say to HTTP/Java Servlet/Web Service stacks.

## 1. Examples

### 1.1. Word Count

The first example is the [`wordCount`](http://github.com/thomasWeise/distributedComputingExamples/tree/master/hadoop/wordCount) project, which is based on the famous and well-known [word counting](http://wiki.apache.org/hadoop/WordCount) Map-Reduce example in the version provided by Luca Menichetti [meniluca@gmail.com](mailto:meniluca@gmail.com) under the GNU General Public License version 2. The original version of the example is nicely discussed in [this blog entry](https://nosqlnocry.wordpress.com/2015/03/13/hadoop-mapreduce-wordcount-example-in-java-introduction-to-hadoop-job/). Further, similar explanations are given [here](http://cs.smith.edu/dftwiki/index.php/Hadoop_Tutorial_1_--_Running_WordCount) and [here](http://wiki.apache.org/hadoop/WordCount).

This example takes in one or multiple text files in folder `input` and produces a word count overview in the folder `output`. The paths to these two folders are its command line parameters. Here we use some older version of the description of this course as input. The [mapper](http://github.com/thomasWeise/distributedComputingExamples/tree/master/hadoop/wordCount/src/main/java/wordCount/WordCountMapper.java) will receive tuples of the form.

    <Integer (line number), Text (the line contents)>

The mapper will split each line into words (ignoring punctuation marks). For each word of the like, it will emit a tuple

    <Text (the word), WriteableInteger (always value 1)>
    
If a word occurs multiple times in a line, one token is emitted for each occurrence. After this mapping step, the [reducer](http://github.com/thomasWeise/distributedComputingExamples/tree/master/hadoop/wordCount/src/main/java/wordCount/WordCountReducer.java) will receive tokens such as

    <Text (the word), Iterable<WriteableInteger> (number of occurrences)>
    
Hadoop as put all the `WriteableInteger` generated by the mapping step which belong to the same `Text (the word)` key into an `Iterable` list for us. Thus, for each word that the mapper has discovered, we get a list with numbers. All we have to do is to add them up and emit tuples of the form:

    <Text (the word), WriteableInteger (total number of occurrences)>
    
The reducer here also acts as combinator, meaning that a reduction step is also performed locally before the results of each mapper is sent to the central reduction step. This way we can already add up some word counts locally and the amount of data that needs to be sent to the central reducer decreases, as two tuples for the same word are already merged. This is possible in this simple form because the output of the reducer is the same as the output of the mapper, just that the `WriteableInteger` part will not necessarily have value `1` afterwards. 

After the reduction step, we therefore know how often each word occurred in the text. Furthermore, since the tuples are sorted automatically before reduction, the word/occurrences list is also nicely sorted alphabetically.

1. [`WordCountDriver`](http://github.com/thomasWeise/distributedComputingExamples/blob/master/hadoop/wordCount/src/main/java/wordCount/WordCountDriver.java)
1. [`WordCountMapper`](http://github.com/thomasWeise/distributedComputingExamples/blob/master/hadoop/wordCount/src/main/java/wordCount/WordCountMapper.java)
1. [`WordCountReducer`](http://github.com/thomasWeise/distributedComputingExamples/blob/master/hadoop/wordCount/src/main/java/wordCount/WordCountReducer.java)

### 1.2. Web Finder

The second example, [`webFinder`](http://github.com/thomasWeise/distributedComputingExamples/tree/master/hadoop/webFinder) tries to use Map-Reduce to find the interconnections between different web sites. It therefore accepts a list of URLs as input, i.e., one or multiple text file(s) where each line represents a single URL. The program has three parameters, the input folder, the output folder, and the maximum depth limit for [spidering](https://en.wikipedia.org/wiki/Web_crawler) the websites. This limit is an optional parameter and 1 by default, meaning that for each base URL, one level of links is followed. The [mapper](http://github.com/thomasWeise/distributedComputingExamples/tree/master/hadoop/wordCount/src/main/java/wordCount/WebFinderMapper.java) receives tuples of type

    <Integer (line number), Text (the line contents, i.e., the URL)>

Each URL is loaded recursively, similar to how a [bot](https://en.wikipedia.org/wiki/Web_crawler) from a search engine would do it. When reading a web page (using Java's URLConnection object), we look for links/usages of other resources, such as images, [CSS](https://en.wikipedia.org/wiki/Cascading_Style_Sheets), [JavaScript](https://en.wikipedia.org/wiki/JavaScript), [links](https://en.wikipedia.org/wiki/Hyperlink), [frames](https://en.wikipedia.org/wiki/Framing_%28World_Wide_Web%29), and [iframes](https://en.wikipedia.org/wiki/Iframes). The latter three are traced recursively up to a given maximum depth (the third, optional, command line parameter of the program). For each linked resource, the mapper will output a tuple

    <Text (with the URL to the resource), Text (with base URL from the input>
    
This means that the [reducer](http://github.com/thomasWeise/distributedComputingExamples/tree/master/hadoop/wordCount/src/main/java/wordCount/WebFinderReducer.java) receives elements of the form

    <Text (with the URL to the resource), Iterable<Text> (all base URLs linking to the resource>

The reduction step is very simple: All input pairs which just contain a single element in the values, i.e., which stand for resources only linked from a single on of the originally specified URLs, are discarded. This leaves only resources linked from multiple input URLs. These resources are thus shared amongst different sites. The reducer returns tuples of

    <Text (with the URL to the shared resource), List<Text> (with all originally specified sites linking to the shared resource)>
    
This will help us to understand how different websites are connected and which resources are vital, i.e., which dependencies are needed by several sites to work properly. We can test this program (see point 2.5) with an example input as shown below and maximum depth `1`. Running the program will take some time.

    http://www.bing.com
    http://www.tudou.com
    http://www.youku.com
    http://www.qq.com
    http://www.baidu.com
    http://www.sogou.com
    
The final output (stored in `output/part-r-00000` and obtained via `bin/hdfs dfs -copyToLocal output/part-r-00000 ./list.txt` for the above input and maximum depth `1`, at the date of this writing, is this:

    http://c.youku.com/aboutcn/youtu  [http://www.tudou.com, http://www.youku.com]
    http://c.youku.com/abouteg/youku  [http://www.tudou.com, http://www.youku.com]
    http://c.youku.com/abouteg/youtu  [http://www.tudou.com, http://www.youku.com]
    http://cbjs.baidu.com/js/m.js [http://www.baidu.com, http://www.qq.com]
    http://css.tudouui.com/skin/__g/img/sprite.gif  [http://www.tudou.com, http://www.youku.com]
    http://events.youku.com/global/scripts/jquery-1.8.3.js  [http://www.tudou.com, http://www.youku.com]
    http://events.youku.com/global/scripts/youku.js [http://www.tudou.com, http://www.youku.com]
    http://images.china.cn/images1/ch/appxz/2.jpg [http://www.qq.com, http://www.youku.com]
    http://images.china.cn/images1/ch/appxz/3.jpg [http://www.qq.com, http://www.youku.com]
    http://js.tudouui.com/v3/dist/js/lib_6.js [http://www.tudou.com, http://www.youku.com]
    http://mail.qq.com  [http://www.baidu.com, http://www.qq.com]
    http://minisite.youku.com/mini_common/urchin.js [http://www.tudou.com, http://www.youku.com]
    http://player.youku.com/jsapi [http://www.tudou.com, http://www.youku.com]
    http://qzone.qq.com [http://www.baidu.com, http://www.qq.com]
    http://res.mfs.ykimg.com/051000004D92DF6197927339BA04E210.js  [http://www.tudou.com, http://www.youku.com]
    http://static.youku.com/user/img/avatar/80/5.jpg  [http://www.tudou.com, http://www.youku.com]
    http://static.youku.com/user/img/avatar/80/9.jpg  [http://www.tudou.com, http://www.youku.com]
    http://weibo.com  [http://www.baidu.com, http://www.qq.com]
    http://www.12377.cn [http://www.baidu.com, http://www.qq.com, http://www.youku.com]
    http://www.12377.cn/node_548446.htm [http://www.qq.com, http://www.youku.com]
    http://www.bjjubao.org  [http://www.baidu.com, http://www.youku.com]
    http://www.china.com.cn/player/video.js [http://www.qq.com, http://www.youku.com]
    http://www.ellechina.com  [http://www.qq.com, http://www.youku.com]
    http://www.hao123.com [http://www.baidu.com, http://www.qq.com]
    http://www.hd315.gov.cn/beian/view.asp?bianhao=010202006082400023 [http://www.tudou.com, http://www.youku.com]
    http://www.miibeian.gov.cn  [http://www.qq.com, http://www.tudou.com, http://www.youku.com]
    http://www.miibeian.gov.cn/publish/query/indexFirst.action  [http://www.tudou.com, http://www.youku.com]
    http://www.pclady.com.cn  [http://www.baidu.com, http://www.qq.com]
    http://www.qq.com [http://www.baidu.com, http://www.qq.com]
    http://www.shjbzx.cn  [http://www.qq.com, http://www.tudou.com]
    http://www.tudou.com  [http://www.tudou.com, http://www.youku.com]
    http://www.tudou.com/about/cn [http://www.tudou.com, http://www.youku.com]
    http://www.tudou.com/about/en [http://www.tudou.com, http://www.youku.com]
    http://www.youku.com  [http://www.baidu.com, http://www.tudou.com, http://www.youku.com]
    http://www.youku.com/show_page/id_z8dc3fdeedcb911e3a705.html  [http://www.tudou.com, http://www.youku.com]
    http://y.qq.com [http://www.baidu.com, http://www.qq.com]
    https://www.alipay.com  [http://www.baidu.com, http://www.youku.com]

In other words, we can see that some of the top sites in China link to each other, as done massively by [tudou](http://www.tudou.com) and [youku](http://www.youku.com). Several pages  also share common resources: [qq](http://www.qq.com) and [youku](http://www.youku.com) both use [http://www.china.com.cn/player/video.js](http://www.china.com.cn/player/video.js) and [http://images.china.cn/images1/ch/appxz/2.jpg](http://images.china.cn/images1/ch/appxz/2.jpg), for instance. If we had added more links to check or increased the search depth, we would probably have found more relationships.

By the way, we also find lots of dodgy links which lead to nowhere or contain illegal characters, causing some warnings to be logged by our system.

1. [`WebFinderDriver`](http://github.com/thomasWeise/distributedComputingExamples/blob/master/hadoop/wordCount/src/main/java/webFinder/WebFinderDriver.java)
1. [`WebFinderMapper`](http://github.com/thomasWeise/distributedComputingExamples/blob/master/hadoop/wordCount/src/main/java/webFinder/WebFinderMapper.java)
1. [`WebFinderReducer`](http://github.com/thomasWeise/distributedComputingExamples/blob/master/hadoop/wordCount/src/main/java/webFinder/WebFinderReducer.java)

## 2. Building and Deployment

### 2.1. Import Project into Eclipse

If you import one of the example projects in this folder in [Eclipse](http://www.eclipse.org), it may first show you a lot of errors. (I recommend using Eclipse Mars or later.) These projects are Maven projects, so you should "update" them first in Eclipse by doing the following. Let's say you want to import the `wordCount` project:

1. Make sure that you can see the `package view` on the left-hand side of the Eclipse window.
2. Right-click on the project (`wordCount`) in the `package view`.
3. In the opening pop-up menu, left-click on `Maven`.
4. In the opening sub-menu, left-click on `Update Project...`.
5. In the opening window...
  1. Make sure the project (`wordCount`) is selected.
  2. Make sure that `Update project configuration from pom.xml` is selected.
  3. You can also select `Clean projects`.
  4. Click `OK`.
6. Now the structure of the project in the `package view` should slightly change, the project will be re-compiled, and the errors should disappear.


### 2.2. Build Project in Eclipse


Now you can actually build the imported project(s), i.e., generate a [`jar`](https://en.wikipedia.org/wiki/JAR_%28file_format%29) file that you can pass to Hadoop. Let's say you want to build the `wordCount` project.

1. Make sure that you can see the `package view` on the left-hand side of the Eclipse window.
2. Right-click on the project (`wordCount`) in the `package view`.
3. In the opening pop-up menu, choose `Run As`.
4. In the opening sub-menu choose `Run Configurations...`.
5. In the opening window, choose `Maven Build`
6. In the new window `Run Configurations` / `Create, manage, and run configurations`, choose `Maven Build` in the small white pane on the left side.
7. Click `New launch configuration` (the first symbol from the left on top of the small white pane).
8. Write a useful name for this configuration in the `Name` field. You can use this configuration again later.
9. In the tab `Main` enter the `Base directory` of the project, this is the folder called `hadoop/wordCount` containing the Eclipse/Maven project.
10. Under `Goals`, enter `clean compile package`. This will build a `jar` archive.
11. Click `Apply`
12. Click `Run`
13. The build will start, you will see its status output in the console window.
14. The folder `target` will contain a file `wordCount-full.jar` after the build. This is the executable archive with our application.


### 2.3. Building under Linux without Eclipse

Under Linux, you can also simply run `make_linux.sh` in this project's folder to build the servlet without Eclipse, given that you have Maven installed.

### 2.4. Setting Up a Single-Node Hadoop Cluster

In order to test our example, we now need to set up a single-node Hadoop cluster. We therefore follow the guide given at [http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html). Here we provide the installation guide for Hadoop 2.7.2 Linux / Ubuntu.

<h4>2.4.1. Download, Unpacking, and Setup</h4>

Here we discuss how to download and unpack Hadoop. 
<ol>
<li>Install prerequisites by running <code>sudo apt-get install ssh rsync</code>.</li>
<li>Go into a base folder where you want to install Hadoop. Let's call this folder <code>X</code>.</li>
<li>Download Hadoop from one of the mirrors provided at <a href="http://www.apache.org/dyn/closer.cgi/hadoop/common/">http://www.apache.org/dyn/closer.cgi/hadoop/common/</a>. I choose <a href="http://www-eu.apache.org/dist/hadoop/common/">http://www-eu.apache.org/dist/hadoop/common/</a> and from there <a href="http://www-eu.apache.org/dist/hadoop/common/hadoop-2.7.2/">hadoop-2.7.2</a> from where I download <a href="http://www-eu.apache.org/dist/hadoop/common/hadoop-2.7.2/hadoop-2.7.2.tar.gz">hadoop-2.7.2.tar.gz</a> into <code>X</code>. If you chose a different Hadoop version, replace <code>2.7.2.</code> accordingly in the following steps.</li>
<li>Once <a href="http://www-eu.apache.org/dist/hadoop/common/hadoop-2.7.2/hadoop-2.7.2.tar.gz">hadoop-2.7.2.tar.gz</a> has fully been downloaded, I either can do <code>Extract Here</code> in the file explorer or <code>tar -xf hadoop-2.7.2.tar.gz</code> in the terminal window to extract the archive.</li>
<li>A new folder named <code>X/hadoop-2.7.2</code> should have appeared. If you chose a different Hadoop version, replace <code>2.7.2.</code> accordingly in the following steps.</li>
<li>In order to run Hadoop, you  must have <code>JAVA&#95;HOME</code> set correctly. Open the file <code>X/etc/hadoop/hadoop-env.sh</code>. Find the line <code>export JAVA&#95;HOME=${JAVA&#95;HOME}</code> and replace it with <code>export JAVA&#95;HOME=$(dirname $(dirname $(readlink -f $(which javac))))</code>.</li></ol>

<h4>2.4.2. Testing basic Functionality</h4>

We can now test whether everything above has turned out well and all is downloaded, unpacked, and set up correctly.
<ol>
<li>In the terminal, enter <code>X/hadoop-2.7.2/</code> and execute the command <code>bin/hadoop</code>. It should display some help and command line options.</li>
<li>We can further test whether Hadoop works by running the single-node example from the <a href="http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html">tutorial</a>. Therefore, in your terminal enter
<pre>
mkdir input
cp etc/hadoop/*.xml input
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar grep input output 'dfs[a-z.]+'
cat output/*
</pre>
This third command should produce a lot of logging output and the last one should say something like <code>1 dfsadmin</code>. If that is the case, you are doing well.
</li></ol>

<h4>2.4.3. Setup for Single-Computer Pseudo-Distributed Execution</h4>

For really using Hadoop in a pseudo-distributed fashion on our local computer, we have to do <a href="http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html#Pseudo-Distributed&#95;Operation">more</a>:
<ol>
<li>Enter the directory <code>X/hadoop-2.7.2/etc</code> in order to create the basic <a href="http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html#Configuration">configuration</a>.</li>
<li>Open the file <code>core-site.xml</code> in the text editor. It should exist, if not, there is something wrong. Try your best by creating it. Remove everything in the file and store the following text, then save and close the file. In other words, the complete contents of the file should become:
<pre>
&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;?xml-stylesheet type="text/xsl" href="configuration.xsl"?&gt;
&lt;configuration&gt;
    &lt;property&gt;
        &lt;name&gt;fs.defaultFS&lt;/name&gt;
        &lt;value&gt;hdfs://localhost:9000&lt;/value&gt;
    &lt;/property&gt;
&lt;/configuration&gt;
</pre></li>
<li>Open the file <code>hdfs-site.xml</code> in the text editor. It should exist, if not, there is something wrong. Try your best by creating it. Remove everything in the file and store the following text, then save and close the file. In other words, the complete contents of the file should become:
<pre>
&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;?xml-stylesheet type="text/xsl" href="configuration.xsl"?&gt;
&lt;configuration&gt;
    &lt;property&gt;
        &lt;name&gt;dfs.replication&lt;/name&gt;
        &lt;value&gt;1&lt;/value&gt;
    &lt;/property&gt;
&lt;/configuration&gt;
</pre></li></ol>

<h4>2.4.4. Setup for SSH for Passwordless Connection to Local Host</h4>

In order to run Hadoop in a pseudo-distributed fashion, we need to enable passwordless SSH connections to the local host. 

<ol>
<li>In the terminal, execute <code>ssh localhost</code> to test if you can open a <a href="https://en.wikipedia.org/wiki/Secure&#95;Shell">secure shell</a> connection to your current, local computer <a href="http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html#Setup&#95;passphraseless&#95;ssh">without needing a password</a>.
</li>
<li>It may say something like:
<pre>ssh: connect to host localhost port 22: Connection refused</pre>
If it does say this (i.e., you did not install the pre-requisites&hellip;), then do
<pre>sudo apt-get install ssh</pre>
and it may say something like
<pre>
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following extra packages will be installed:
  libck-connector0 ncurses-term openssh-server openssh-sftp-server
  ssh-import-id
Suggested packages:
  rssh molly-guard monkeysphere
The following NEW packages will be installed:
  libck-connector0 ncurses-term openssh-server openssh-sftp-server ssh
  ssh-import-id
0 upgraded, 6 newly installed, 0 to remove and 0 not upgraded.
Need to get 661 kB of archives.
After this operation, 3,528 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
...
Setting up ssh-import-id (4.5-0ubuntu1) ...
Processing triggers for ufw (0.34-2) ...
Setting up ssh (1:6.9p1-2ubuntu0.2) ...
Processing triggers for libc-bin (2.21-0ubuntu4.1) ...
Processing triggers for systemd (225-1ubuntu9.1) ...
Processing triggers for ureadahead (0.100.0-19) ...
</pre>
OK, now you've got SSH installed. Do <code>ssh localhost</code> again.</li>
<li>It may ask you something like 
<pre>
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:HZUVFF77GAh5cF/sg8YhjRf1gSGJ9ui5ksdf2GAl5Ha.
Are you sure you want to continue connecting (yes/no)? 
</pre>
If it does ask you this, just type <code>yes</code> and hit enter (it may then say something like <code>Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.</code>). If it does not ask you this, it does not matter.</li>
<li>
The important thing is the next step: IF it asks you something like <code>xyz@localhost's password:</code>, hit <code>Ctrl-C</code> and do the things below. Otherwise, you can directly skip to the next point 2.4.5. So, If you were asked for a password, enter the following into your terminal:
<pre>
ssh-keygen -t dsa -P '' -f ~/.ssh/id&#95;dsa
cat ~/.ssh/id&#95;dsa.pub &gt;&gt; ~/.ssh/authorized&#95;keys
chmod 0600 ~/.ssh/authorized&#95;keys 
</pre></li>
<li>You will get displayed some text such as <code>Generating public/private dsa key pair.</code> followed by a couple of other things. After completing the above commands, you should test the result by again executing <code>ssh localhost</code>. You will now no longer be asked for a password and directly receive a welcome message, something like <code>Welcome to Ubuntu 15.10 (GNU/Linux 4.2.0-35-generic x86&#95;64)</code> or whatever Linux distribution you use. Via a ssh connection, you can, basically, open a terminal to and run commands on a remote computer (which, in this case, is your own, current computer). You can return to the normal (non-ssh) terminal by entering <code>exit</code> and pressing return, after which you will be notified that <code>Connection to localhost closed.</code></li>
</ol>

<h4>2.4.6. Running the Hadoop-Provided Map-Reduce Job Locally</h4>

We now want to test whether our installation and setup works correctly by further following the steps given in the <a href="http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html#Execution">tutorial</a>.
<ol>
<li>Format the HDFS file system by entering <code>bin/hdfs namenode -format</code> followed by return. You will receive a lot of log output.</li>
<li>Start the <code>NameNode</code> and <code>DataNode</code> daemons by running <code>sbin/start-dfs.sh</code>. You may get some logging output messages, which *may* be followed by something like
<pre>
The authenticity of host '0.0.0.0 (0.0.0.0)' can't be established.
ECDSA key fingerprint is SHA256:HZUVFF77GAh5cF/sg8YhjRf1gSGJ9ui5ksdf2GAl5Ha.
Are you sure you want to continue connecting (yes/no)? 
</pre>    
which you would answer with <code>yes</code> followed by a hit to the enter button. If, after that, you get a message like <code>0.0.0.0: packet&#95;write&#95;wait: Connection to 127.0.0.1: Broken pipe</code>, enter <code>sbin/stop-dfs.sh</code>, hit return, and do <code>sbin/start-dfs.sh</code> again.</li>
<li>In your web browser, open <code>http://localhost:50070/</code>. It should display a web page giving an overview about the Hadoop system now running on your local computer.</li>
<li>Now we can setup the required stuff for the example jobs (making HDFS directories and copying the input files). Make sure to replace <code>&lt;userName&gt;</code> with your user/login name on your current machine.
<pre>
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/&lt;userName&gt;
bin/hdfs dfs -put etc/hadoop input
</pre></li>
<li>We can now run the job via
<pre>
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar grep input output 'dfs[a-z.]+'
</pre></li>
<li>We obtain the output of the job via
<pre>
bin/hdfs dfs -get output output
cat output/*
</pre></li>
<li>Like in the local test from point 2.4.2., after a lot of log output, we should sett something like <code>1 dfsadmin</code>.</li>
<li>Finally, we need to shutdown Hadoop by running <code>sbin/stop-dfs.sh</code></li>
</ol> 

### 2.5. Running a Compiled Example project

We now want to run one of the provided examples. Let us assume we want to run the <code>wordCount</code> example. For other examples, just replace <code>wordCount</code> with their names in the following text. I assume that the <code>distributedComputingExamples</code> repository is located in a folder <code>Y</code> on your machine.
<ol>
<li>Open a terminal and enter your Hadoop installation folder. I assume you installed Hadoop version <code>2.7.2</code> into a folder named <code>X</code>, so you would <code>cd</code> into <code>X/hadoop-2.7.2/</code>.</li>
<li>We want to start with a "clean" file system, so let us repeat some of the setup steps. Don't forget to replace <code>&lt;userName&gt;</code> with your local login/user name.
<pre>
bin/hdfs namenode -format
</pre>
(answer with <code>Y</code> when asked whether to re-format the file system)
<pre>
sbin/start-dfs.sh
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/&lt;userName&gt;
</pre>
If you actually properly cleaned up the file system after running your last examples (see the second-to-last step here), you just need to do <code>sbin/start-dfs.sh</code> and do not need to format the HDFS.</li>
<li>Copy the input data of the example into HDFS. You find this data in the example folder <code>Y/distributedComputingExamples/hadoop/wordCount/input</code>. So you will perform <code>bin/hdfs dfs -put Y/distributedComputingExamples/hadoop/wordCount/input input</code>. Make sure to replace <code>Y</code> with the proper path. If copying fails, go to "2.6. Troubleshooting".</li>
<li>Do <code>bin/hdfs dfs -ls input</code> to check if the files have properly been copied.</li>
<li>You can now do <code>bin/hadoop jar Y/distributedComputingExamples/hadoop/wordCount/target/wordCount-full.jar input output</code>. This command will start the main class of the example, which resides in the fat jar <code>wordCount-full.jar</code>, with the parameters <code>input</code> and <code>output</code>. <code>input</code> here is the input folder, which we previously have copied to the Hadoop file system. <code>output</code> is the output folder to be created. If you execute this command, you will see lots of logging information.</li>
<li>Do <code>bin/hdfs dfs -ls output</code>. You will see output like
<pre>
Found 2 items
-rw-r--r--   1 tweise supergroup          0 2016-04-22 18:48 output/&#95;SUCCESS
-rw-r--r--   1 tweise supergroup        303 2016-04-22 18:48 output/part-r-00000
</pre></li>
<li>You can read the results via <code>bin/hdfs dfs -cat output/part-r-00000 | less</code> which will result - in the case of the <code>wordCount</code> example - in something like
<pre>
A       1
API     4
Actually        2
All     1
Apache  1
As      2
Axis    1
Based   1
Both    1
By      1
C       2
CSS     2
Calls   1
Cascading       1
Communication   1
Control 2
Datagram        1
Description     2
Each    1
Everything      2
Extensible      1
Finally 1
For     3
</pre></li>
<li>You can download the result file to your local folder via <code>bin/hdfs dfs -copyToLocal output/part-r-00000 .</code>. Now you will find the text file with the results in the current folder, i.e., <code>X/hadoop-2.7.2/</code>.</li>
<li>After being done, you should clean up the file system by deleting the <code>input</code> and <code>output</code> folder in the HDFS (not the original input folder of the project!). This way, you do not need to format the HDFS for the next example. Anyway, you do:
<pre>
bin/hdfs dfs -rm -R input
bin/hdfs dfs -rm -R output    
</pre></li>
<li>Finally, shut down the system by calling <code>sbin/stop-dfs.sh</code>.</li>
</ol>

### 2.6 Troubleshooting

<h4>2.6.1. "No such file or directory"</h4>

Sometimes, you may try to copy some file or folder to HDFS and get an error that no such file or directory exists. Then do the following: 

<ol>
<li>Execute <code>sbin/stop-dfs.sh</code></li>
<li>Delete the folder <code>/tmp/hadoop-&lt;userName&gt;</code>, where <code>&lt;userName&gt;</code> is to replaced with your local login/user name.</li>
<li>Now perform
<pre>
bin/hdfs namenode -format 
sbin/start-dfs.sh
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/&lt;userName&gt;
</pre>
</li><li>
If you now repeat the operation that failed before, it should succeed.
</li>
</ol>

## 3. Licensing

Some of the examples take some inspiration from the <a href="https://github.com/H4ml3t/maven-hadoop-java-wordcount-template">maven-hadoop-java-wordcount-template</a> by <a href="https://github.com/H4ml3t">H3ml3t</a>, for which no licensing information is provided. The examples, are entirely differently in several ways, for instance in the way we build fat jars. Anyway, this original project is nicely described in <a href="https://nosqlnocry.wordpress.com/2015/03/13/hadoop-mapreduce-wordcount-example-in-java-introduction-to-hadoop-job/">this blog entry</a>.

Furthermore, the our `wordCount` is based on the well-known <a href="http://wiki.apache.org/hadoop/WordCount">word counting example</a> for Hadoop's map reduce functionality. It is based on the version by provided Luca Menichetti <a href="mailto:meniluca@gmail.com">meniluca@gmail.com</a> under the GNU General Public License version 2.