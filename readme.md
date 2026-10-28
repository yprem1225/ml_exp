HADOOP MAPREDUCE WORDCOUNT PRACTICAL
EXAM-DAY NOTEBOOK / NOTEPAD GUIDE
Environment: Ubuntu 24.04 in WSL2 | Hadoop 3.4.1 | Java 21
Purpose: Start Hadoop, prepare WordCount, run it, and verify output.

IMPORTANT: Read each step and run commands in order. Do not copy the prompt symbol "$".
This guide assumes your Hadoop installation is:
  /home/prem_yadav/hadoop-3.4.1
and Java is:
  /usr/lib/jvm/java-21-openjdk-amd64

============================================================
0. OPEN UBUNTU
============================================================

On Windows:
  Start menu -> search "Ubuntu 24.04 LTS" -> Open

OR in Windows Terminal / PowerShell:
  wsl

You should see a prompt similar to:
  prem_yadav@premyadav1225:~$

If the user/prompt differs, that is okay. Use your own Linux username.

============================================================
1. START HADOOP SERVICES
============================================================

Run:

  start-dfs.sh
  start-yarn.sh
  jps

In the jps output, look for processes such as:
  NameNode
  DataNode
  SecondaryNameNode
  ResourceManager
  NodeManager
  Jps

Names can vary a little. If services are already running, start scripts may say
they are already running; that is okay.

Do NOT run "hdfs namenode -format" during normal startup.
Formatting can erase HDFS metadata / make existing HDFS data inaccessible.

============================================================
2. MAKE A NEW PRACTICAL WORK FOLDER
============================================================

This uses a fresh folder so it does not overwrite your older files.

  mkdir -p ~/wordcount_exam
  cd ~/wordcount_exam

Check location:
  pwd

Expected:
  /home/prem_yadav/wordcount_exam

============================================================
3. CREATE THE INPUT TEXT FILE
============================================================

Create/edit the input file with Nano:

  nano input.txt

Type this sample input (three lines):

  hello prem
  prem hello
  prem

Save and exit Nano:
  Ctrl + O
  Enter
  Ctrl + X

Check your file:

  cat input.txt

EXPECTED INPUT:
  hello prem
  prem hello
  prem

Expected counts:
  hello = 2
  prem  = 4

============================================================
4. CREATE THE JAVA WORDCOUNT PROGRAM
============================================================

Open the Java source:

  nano WordCount.java

Paste the entire code below. Ensure the class name is WordCount.

------------------------------------------------------------
JAVA SOURCE CODE
------------------------------------------------------------

import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

    public static class TokenizerMapper
            extends Mapper<Object, Text, Text, IntWritable> {

        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public void map(Object key, Text value, Context context)
                throws IOException, InterruptedException {

            StringTokenizer itr = new StringTokenizer(value.toString());

            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());
                context.write(word, one);
            }
        }
    }

    public static class IntSumReducer
            extends Reducer<Text, IntWritable, Text, IntWritable> {

        private IntWritable result = new IntWritable();

        public void reduce(Text key, Iterable<IntWritable> values,
                Context context) throws IOException, InterruptedException {

            int sum = 0;

            for (IntWritable val : values) {
                sum += val.get();
            }

            result.set(sum);
            context.write(key, result);
        }
    }

    public static void main(String[] args) throws Exception {

        if (args.length != 2) {
            System.err.println("Usage: WordCount <input path> <output path>");
            System.exit(2);
        }

        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");

        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setCombinerClass(IntSumReducer.class);
        job.setReducerClass(IntSumReducer.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}

------------------------------------------------------------

Save and exit:
  Ctrl + O
  Enter
  Ctrl + X

============================================================
5. COMPILE JAVA AND CREATE JAR
============================================================

Make sure you are in the practical folder:

  cd ~/wordcount_exam

Create a classes folder:

  mkdir -p classes

Compile:

  javac -classpath "$(hadoop classpath)" -d classes WordCount.java

If compilation succeeds, it usually returns to the prompt with no message.

Create the JAR:

jar -cvf wordcount.jar -C classes .

Check files:

  ls

You should see:
  WordCount.java
  input.txt
  classes
  wordcount.jar

============================================================
6. CREATE HDFS INPUT DIRECTORY AND UPLOAD INPUT
============================================================

HDFS is Hadoop's distributed file system. The local input.txt and HDFS input
are separate copies. Whenever you change local input.txt, upload it again.

Create HDFS input directory:

  hdfs dfs -mkdir -p /wordcount/input

Upload/replace the local input file:

  hdfs dfs -put -f input.txt /wordcount/input/

VERIFY HDFS INPUT BEFORE RUNNING:

  hdfs dfs -cat /wordcount/input/input.txt

It MUST show the same content you see from "cat input.txt".
If it does not match, do not run the job yet. Upload again from the correct
local folder.

Note: "/wordcount" is just a chosen folder name in HDFS. It is not the JAR name
and is not your local working directory.

============================================================
7. RUN THE MAPREDUCE WORDCOUNT JOB
============================================================

Remove old output directory if it exists:

  hdfs dfs -rm -r /wordcount/output

If it says "No such file or directory", that is okay on the first run.

Run the job:

  hadoop jar wordcount.jar WordCount /wordcount/input /wordcount/output

Wait for:
  Job job_... completed successfully

Some INFO or WARN messages may appear. The important thing is whether the job
completes successfully. A common command-line parsing warning is not necessarily
a job failure.

============================================================
8. DISPLAY AND VERIFY OUTPUT
============================================================

Run:

  hdfs dfs -cat /wordcount/output/part-r-00000

For the sample input:
  hello prem
  prem hello
  prem

Expected output (word order may vary):

  hello   2
  prem    4

If output shows old words (such as "hadoop" or "world"), check the HDFS input:

  hdfs dfs -cat /wordcount/input/input.txt

Then upload the correct local file again:

  cd ~/wordcount_exam
  hdfs dfs -put -f input.txt /wordcount/input/

Delete output and rerun:

  hdfs dfs -rm -r /wordcount/output
  hadoop jar wordcount.jar WordCount /wordcount/input /wordcount/output
  hdfs dfs -cat /wordcount/output/part-r-00000

============================================================
9. IF YOU GET AN ERROR: QUICK CHECKS
============================================================

A) "Output directory already exists"
Delete the old output first:
  hdfs dfs -rm -r /wordcount/output

B) Output has old words/counts
Check local input:
  cat ~/wordcount_exam/input.txt

Check HDFS input:
  hdfs dfs -cat /wordcount/input/input.txt

Upload the correct local file:
  cd ~/wordcount_exam
  hdfs dfs -put -f input.txt /wordcount/input/

Then delete output and rerun the job.

C) "Input path does not exist"
Check:
  hdfs dfs -ls /wordcount/input
If missing, create and upload:
  hdfs dfs -mkdir -p /wordcount/input
  cd ~/wordcount_exam
  hdfs dfs -put -f input.txt /wordcount/input/

D) Java source compile error
Confirm:
  java -version
  javac -version
  hadoop version
Make sure the source file is named exactly:
  WordCount.java
and its public class is:
  public class WordCount

E) NameNode / ResourceManager connection issue
Start services again:
  start-dfs.sh
  start-yarn.sh
Check:
  jps

F) Do not format NameNode as a routine fix.
Never run this as a normal startup step:
  hdfs namenode -format

============================================================
10. EXAM QUICK RUN (AFTER YOU HAVE CREATED JAVA/JAR BEFORE)
============================================================

If source and JAR already exist, after opening Ubuntu:

  start-dfs.sh
  start-yarn.sh
  cd ~/wordcount_exam
  cat input.txt
  hdfs dfs -mkdir -p /wordcount/input
  hdfs dfs -put -f input.txt /wordcount/input/
  hdfs dfs -cat /wordcount/input/input.txt
  hdfs dfs -rm -r /wordcount/output
  hadoop jar wordcount.jar WordCount /wordcount/input /wordcount/output
  hdfs dfs -cat /wordcount/output/part-r-00000

If you changed WordCount.java, compile and rebuild the JAR again:
  javac -classpath "$(hadoop classpath)" -d classes WordCount.java
  jar -cvf wordcount.jar -C classes .

============================================================
11. VIVA: WHAT EACH PART DOES
============================================================

1. Mapper:
   Reads input lines, splits them into words, and emits (word, 1).

2. Combiner:
   Performs local addition of word counts before sending data to the Reducer.
   It is an optional optimization.

3. Reducer:
   Receives each word and its list of values, adds them, and emits (word, total).

4. HDFS:
   Hadoop Distributed File System. Stores the input and output files.

5. Job:
   The MapReduce task configured and submitted to Hadoop/YARN.

6. JAR:
   A Java archive containing compiled program classes.

7. Why remove output before every run?
   Hadoop expects the output directory not to exist before the job starts.

8. Why upload input after editing it?
   Local files and HDFS files are separate. Editing local input does not update HDFS.

============================================================
FINAL CHECKLIST
============================================================

[ ] Open Ubuntu
[ ] Start DFS and YARN
[ ] Go to ~/wordcount_exam
[ ] Check input.txt
[ ] Compile Java (if needed)
[ ] Build JAR (if needed)
[ ] Upload input.txt to HDFS
[ ] Verify HDFS input with hdfs dfs -cat
[ ] Remove old HDFS output
[ ] Run MapReduce job
[ ] Display part-r-00000
[ ] Confirm expected counts

Good luck in your practical exam!
