---
layout: post
title: checker
author: MaskRay
tags: [algorithm, oi]
---

```python
#!/usr/bin/env python
#
#   Copyright (C) 2010, 2011 MaskRay
#
#   This program is free software: you can redistribute it and/or modify
#   it under the terms of the GNU General Public License as published by
#   the Free Software Foundation, either version 3 of the License, or
#   (at your option) any later version.
#
#   This program is distributed in the hope that it will be useful,
#   but WITHOUT ANY WARRANTY; without even the implied warranty of
#   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#   GNU General Public License for more details.
#
#   You should have received a copy of the GNU General Public License
#   along with this program.  If not, see <http://www.gnu.org/licenses/>.
#
#   ------------------------------------------------------------------------------
#
#   checker
#           check programs using testdata

COPYING = '''checker
Copyright (C) 2010, 2011 MaskRay
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
'''

def rreplace(s1, s2, s3):
    p = s1.rfind(s2)
    return s1[:p] + s3 + s1[p+len(s2):] if p != -1 else s1

def test(s, pat1, pat2):
    p = s.find(pat1)
    if p == -1:
        return False
    q = s.find(pat2)
    while q != -1:
        if not p <= q < p+len(pat1):
            return True
        q = s.find(pat2, q+len(pat2))
    return False

if __name__ == '__main__':

    import math, sys, os, re, subprocess, threading, time, tempfile
    from optparse import OptionParser

    def main():

        def print_copying():
            sys.stdout.write(COPYING)
            exit(0)

        usage = '''usage: %prog source [input] [output]
        source  - source file
        input   - suffix of input files (default `in')
        output  - suffix of output files (default `out')'''
        parser = OptionParser(usage=usage)
        parser.add_option('-p', '--path', action='store',
                          dest='path', default='/tmp',
                          help='specify the path of testdata')
        parser.add_option('-i', '--dataid', action='store',
                          dest='dataid', default='', help='specify the data id')
        parser.add_option('-d', '--detail', action='store_true',
                          dest='show_detail', default=False, help='show details')
        parser.add_option('-s', '--showtime', action='store_true',
                          dest='show_time', default=False, help='show time used')
        parser.add_option('-t', '--time-limit', action='store', type='float',
                          dest='time_limit', default=1.)
        parser.add_option('--version', action="callback",
                          callback=lambda o, s, v, p: print_copying(),
                          help="print the COPYING and exit")
        (options, args) = parser.parse_args()

        options.path = os.path.expanduser(options.path)
        if len(args) < 1 or len(args) > 4:
            parser.print_usage()
            sys.exit(1)
        if len(args) < 2:
            args.append('in')
        if len(args) < 3:
            args.append('out')
        r = re.compile('\d+')

        L = [file for file in os.listdir(options.path) if test(file, args[0], args[1])]
        for file in sorted(L, key = lambda x: int(r.findall(x)[-1] if
            r.findall(x) else sys.maxint)):
            if options.dataid != '' and (not r.findall(file) or
                    r.findall(file)[-1] != options.dataid):
                continue
            print('---' + file + '---')

            temp = tempfile.NamedTemporaryFile()
            sub = subprocess.Popen('./main', stdin=open(os.path.join(options.path, file)), stdout=temp)
            bgn = time.time()
            timer = threading.Timer(options.time_limit, sub.terminate)
            timer.start()
            ret = sub.wait()
            end = time.time()
            timer.cancel()

            RESET = '\033[0m'
            if ret == -11:
                print('\033[1;34m' + 'runtime error' + RESET)
            elif ret == -15:
                print('\033[1;35m' + 'time limit exceed' + RESET)
            elif ret == 0:
                sys.stdout.write('\033[1m')
                sys.stdout.flush()
                with open(os.path.join(options.path, rreplace(file, args[1], args[2]))) as f:
                    lines = f.readlines()

                if len(lines) == 1 and re.search('''\d+\.\d+''', lines[0]):
                    res = float(lines[0])
                    with open(temp.name) as f:
                        lines = f.readlines()
                    if len(lines) != 1:
                        print('wrong answer' + RESET)
                    elif math.isnan(float(lines[0])) or abs(float(lines[0])-res) > 1e-9:
                        print('< %f\n---\n> %f\n' % (res, float(lines[0])) + RESET)
                elif 0 == os.system('diff --strip-trailing-cr %s %s %s' %
                                  ('' if options.show_detail else '-q',
                                   os.path.join(options.path, rreplace(file, args[1], args[2])),
                                   temp.name)) and options.show_time:
                    print(RESET + '%.3f second(s)' % (end-bgn))
                else:
                    sys.stdout.write(RESET)
                    sys.stdout.flush()
            else:
                print('\033[1;32m' + 'exit abnormally' + RESET)

    main()
```
