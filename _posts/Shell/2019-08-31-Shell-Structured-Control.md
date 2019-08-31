# structured control

## if

    if [ conditon1 ] && [ condtition2 ] || [ condtion3 ]; then
        command1
    else
        command2
    fi

### numeric comparisons

- $var1 -eq $var2
- $var1 -ge $var2
- $var1 -gt $var2
- $var1 -le $var2
- $var1 -lt $var2
- $var1 -ne $var2

### string comparisons

- $str1 = $str2
- $str1 != $str2
- $str1 \\< $str2
- $str1 \\> $str2
- -n $str1
- -z $str1

### file comparisons

- -e file \/\/ existed
- -s file \/\/ existed and not empty file or directory (but directory is never empty)
- -d file \/\/ existed and is directory
- -f file \/\/ existed and is file
- -r file \/\/ existed and readable
- -w file \/\/ existed and writable
- -x file \/\/ existed and excutable
- -O file \/\/ existed and owned by user
- -G file \/\/ existed and owned by group
- file1 -nt file2 \/\/ file1 is newer than file2
- file1 -ot file2 \/\/ file1 is older than file2

### advanced if

    if (( formulas )); then
        command
    fi

    if (( $var ** 2 > 90 )); then
        command
    fi


    if [[ pattern matching ]]; then
        command
    fi

    if [[ $USER == r* ]]; then
        command
    fi

### case statement

    case variable in
    pattern1 | pattern2) command1;;
    pattern3) command2;;
    *) default commands;;
    esac



## for

    for var in list; do
        command
    done

    for var in $(cat "./file"); doc
        echo $var
    done

### change the separator

    OLD_IFS=$IFS
    IFS=$'\n'
    for var in $(cat "./file"); doc
        echo $var
    done
    IFS=$OLD_IFS

### C-style for loop

    for (( initial statement; condition; iteration process )); do
        command
    done

### while statement

*while* runs the loop when the condition is true;
*util* runs then loop util the condition is true.

    while [ condition ]; do
        command
    done

    util [ conditon ]; do
        command
    done

### loop control

    break
    break n // break outer n loop

    continue
    continue n // continue n outer loop

### redirection loop output

output of loop can be redirected or piped with '|' '>' '>>' operator.
