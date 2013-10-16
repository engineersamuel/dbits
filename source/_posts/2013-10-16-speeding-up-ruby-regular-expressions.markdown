---
layout: post
title: "Speeding up Ruby Regular Expressions"
date: 2013-10-16 15:36
comments: true
sharing: true
categories: [ruby, regex, regular expressions, performance]
---

## Introduction

Ruby regular expressions could be called slow if you were to compare them to perl regex or grep or the like.   Performance is somewhat relativistic depending on the exact problem.  However given the option between two duplicate end results, one being faster with minimal to no trade-offs, you'd probably take the latter.

## Problem

Parsing logs in Ruby with Regexp#match can be slow.  You can optimize your regular expression to a degree, and I admit I probably didn't explore that area to the absolute fullest, however at some point you will still deal with the cost of the #match method.

I encountered this problem while writing [splog](https://github.com/engineersamuel/splog). Splog gives you a simple means to parse any file based on yml defined regular expressions.  When developing splog I was first testing with anywhere from a 1 to a 1000 line log file.  After I implemented the initial round of features I stress tested with a 90k line log file.  That is when I encountered an ETA of 4 hours, something had to change.

## Analysis

Let's look at some of the profiling data for ruby regular expressions.  A bit of code first.  To parse the log with a particular regular expression I first must parse the current line then the second line:
{% codeblock lang:ruby %}
m = @pattern.match(line)
{% endcodeblock %}

The reason I have to match the subsequent line is to handle rolling next lines into the current line.  Think java exceptions across multiple log entries.

The problem though is for a 90k line file I am doing at least 180k Regexp#match calls.  This get's expensive fast.  To illustrate here is the `time` and `ruby-prof` of 100 lines of the 90k line server.log:


{% codeblock %}
time head -n 100 /tmp/server.log | ruby -Ilib ./bin/splog --line-count=100 -p jboss_log4j_common -c  -o -v  
11.13s user 0.05s system 99% cpu 11.185 total
{% endcodeblock %}

{% codeblock %}
head -n 100 /tmp/server.log | ruby lib/splog-prof.rb --line-count=100 -p jboss_log4j_common -c /tmp/.splog.yml -o -v
%self      total      self      wait     child     calls  name
 98.84     11.117    11.117     0.000     0.000      298   Regexp#match
  0.07      0.008     0.008     0.000     0.000     9751   Kernel#class
  0.07      0.012     0.008     0.000     0.004      398   Array#each
  0.05      0.006     0.006     0.000     0.000      995   String#inspect
  0.04     11.189     0.005     0.000    11.184      298   Splog::LogParser#parse_line
  0.04      0.022     0.005     0.000     0.017      199   Hash#each
  0.04      0.004     0.004     0.000     0.000      199   MatchData#names
  0.03      0.047     0.004     0.000     0.043     3184  *Splog::LogParser#sigflat
  0.02     11.248     0.003     0.000    11.245        1   Enumerator::Generator#each
  0.02      0.008     0.002     0.000     0.006      199   Array#to_s
  0.02      0.002     0.002     0.000     0.000     1194   Hash#[]=
  0.02      0.010     0.002     0.000     0.008      199   Array#map!
  0.01      0.001     0.001     0.000     0.000      995   MatchData#[]
  0.01      0.001     0.001     0.000     0.000      199   Digest::Base#update
  0.01      0.001     0.001     0.000     0.000      995   Hash#has_key?
  0.01      0.001     0.001     0.000     0.000      199   Enumerator#peek
  0.01      0.003     0.001     0.000     0.002      199   Digest::Instance#digest
  0.01      0.004     0.001     0.000     0.003      199   <Class::Digest::Class>#digest
  0.01      0.052     0.001     0.000     0.051      199   Splog::LogParser#createsig
  0.01      0.001     0.001     0.000     0.000      199   Array#sort!
  0.01      0.055     0.001     0.054     0.000      100   Enumerator::Yielder#<<
  0.00      0.004     0.000     0.000     0.004      199   <Class::Digest::Class>#hexdigest
  0.00      0.000     0.000     0.000     0.000      199   Module#to_s
  0.00      0.000     0.000     0.000     0.000      398   Digest::Base#reset
  0.00      0.000     0.000     0.000     0.000      199   <Class::Digest::Base>#allocate
  0.00      0.000     0.000     0.000     0.000      199   Digest::Base#finish
  0.00      0.000     0.000     0.000     0.000      198   NilClass#nil?
  0.00      0.000     0.000     0.000     0.000      198   Kernel#nil?
  0.00      0.000     0.000     0.000     0.000      199   Digest::Class#initialize
  0.00      0.000     0.000     0.000     0.000      100   Enumerator#next
{% endcodeblock %}

11 seconds for only 100 lines is bad.  Bad enough to lead me to hours of trial and error and attempting to optimize the regex patterns and scouring stack overflow and google for answers.

The best answer I got from hours of research was 'use =~ since #match is slow'.  I tried that and it made no difference since I still needed the matching group data since the named groups are what defines the json fields and what is inserted into Mongo.

After a lot of trial and error and testing out different designs I had to rethink the problem.  Ruby is a joy to code in and implements the syntax of regular expressions just fine.  But Ruby isn't optimized like perl or grep from a performance standpoint.

With that in mind how about let's use a scalpal instead of a hammer for this problem. 

## Solution

Let native grep handle testing all subsequent lines to see if they need to be Regexp#match then let Ruby #match only when really necessary.  Not only did this reduce the #match calls by 50% but it sped up the program by upwards of 1000% percent.

Since grep doesn't support named groups I couldn't use grep everywhere, but that's no worry.

The main change is to take anywhere I match a next line in the enumerator and replace it with system call to grep.  This also required that I change the regular expression to remove the named groups like `?<SomeGroup>` and replace them with an empty string since grep doesn't support named group syntax:

{% codeblock lang:ruby%}
# where parse_line invokes Regexp#match on the next log line to test match forward matches
parsed_next_line = @pattern_match_forward.nil? ? parse_line(next_line) : parse_line(next_line, {:regex => @pattern_match_forward})
{% endcodeblock %}

Instead of resulting in a Regexp#match, just call the Open3.capture3 to get a success or not success on the native match invocation on the pattern

{% codeblock lang:ruby %}
# where @pattern_match_forward is the named group version with the named groups replaced with '' to be compatible with grep
o, e, s = nil
begin
    o, e, s = Open3.capture3(@pattern_match_forward.nil? ? @pattern_egrep : @pattern_match_forward_egrep, :stdin_data=>next_line)
rescue Errno::EPIPE
    puts "Connection broke!"
    nil
end
{% endcodeblock %}

In the above Open3.capture3 return the `s` represents a successful match verified by `s && s.success?`

Now that native grep is being used let's take a look at time and ruby-prof again.
{% codeblock %}
time head -n 100 /tmp/server.log | ruby -Ilib ./bin/splog --line-count=100 -p jboss_log4j_common -c  -o -v  
0.96s user 0.84s system 104% cpu 1.723 total
{% endcodeblock %}

That simple change reduced the time from 11 seconds to 1.7.  The difference is much more dramatic when parsing the whole file.  On my quad core laptop parsing the entire 90k log file with the old code had an ETA of 4 hours while with the new code it was only 18 minutes.

Let's look at the `ruby-prof` of the first 100 lines as well
{% codeblock %}
head -n 100 /tmp/server.log | ruby lib/splog-prof.rb --line-count=100 -p jboss_log4j_common -c /tmp/.splog.yml -o -v
 %self      total      self      wait     child     calls  name
 29.98      0.314     0.303     0.000     0.010      396   <Class::Thread>#new
 26.94      0.273     0.273     0.000     0.000      198   Kernel#spawn
 18.12      0.185     0.183     0.000     0.002      198   <Module::Process>#detach
  2.36      0.896     0.024     0.000     0.873      198   Open3#popen_run
  1.29      0.054     0.013     0.041     0.000      594   Thread#value
  1.22      0.031     0.012     0.018     0.000      198   IO#write
  1.07      0.020     0.011     0.000     0.009      596   Array#each
  1.01      0.010     0.010     0.000     0.000      396   Thread#initialize
  0.87      1.012     0.009     0.000     1.003        1   Enumerator::Generator#each
  0.83      0.008     0.008     0.000     0.000     1188   IO#initialize
  0.82      0.008     0.008     0.000     0.000      100   Regexp#match
  0.77      0.008     0.008     0.000     0.000     1188   IO#close
  0.62      0.016     0.006     0.000     0.010      594   <Class::IO>#pipe
  0.47      0.921     0.005     0.000     0.916      198   Open3#popen3
  0.46      0.931     0.005     0.000     0.926      198   Open3#capture3
  0.36      0.004     0.004     0.000     0.000     4900   Kernel#class
  0.31      0.003     0.003     0.000     0.000      500   String#inspect
  0.28      0.003     0.003     0.000     0.000     1194   Hash#[]=
  0.27      0.046     0.003     0.000     0.043      100   Splog::LogParser#parse_line
{% endcodeblock %}

The beautiful part of this profiling is there are no longer 298 expensive calls to Regexp#match.  There are only now as many calls to Regexp#match as there are lines in the file.  This dramatically speeds the code up.   Offloading the match forward parses to grep definitely pays off here with minimal cost to spawning the grep process.


## Final Thoughts

It wasn't my first or second option to go outside of Ruby to speed up the regular expressions.  I felt it was too much of a hack at first.  If I can make a performance optimization within a framework/library/language versus outside of it and all things are equal, I'd always choose in..  I definitely know there are other optimizations I can make, however once I ruled out all reasonable ones that I could find, this solution presented itself and it is actually a very clean solution.  

I struggled with finding a good solution to the slowness of Ruby Regexp#match on large files, so hopefully this post will save some of you some headaches when searching for options.
