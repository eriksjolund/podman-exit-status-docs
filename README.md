# podman-exit-status-docs

## Introduction

When a process terminates, its parent process can retrieve an integer representing the
[exit status](https://en.wikipedia.org/wiki/Exit_status) of the terminated process.
This _exit status code_ is an integer number between `0` and `255`. The exit status code `0` means that the
process ran successfully. Any other exit status code means that the process failed.

### Example using a shell

1. Start a bash shell
2. Run a program that terminates with exit status `1`
   ```
   bash -c "exit 1"
   ```
3. Check the exit status of the last command
   ```
   echo $?
   ```
   The command prints this output
   ```
   1
   ```

## Podman exit status

Podman terminates with the same exit status as the command that was started in the container.

| command | podman exit status |
| --                            |                --  |
|   podman run alpine sh -c "exit 0" | 0 |
|   podman run alpine sh -c "exit 1" | 1 |
|   podman run alpine sh -c "exit 2" | 2 |
|   podman run alpine sh -c "exit 3" | 3 |
|   podman run alpine sh -c "exit 4" | 4 |
|        ... | ... |
|   podman run alpine sh -c "exit 254" | 254 |
|   podman run alpine sh -c "exit 255" | 255 |

(The lines for 5-253 were left out of the table for brevity)


## Podman exit status when using `--detach`

Podman always terminates with exit status code `0` when the option
__--detach__ is given to the  __podman run__ command.
Podman will in detached mode terminate right
away after starting the container.
Podman does not wait for the termination of the container process
so Podman does not know of any container command _exit status code_.

| command | podman exit status |
| --                            |                --  |
|   podman run --detach alpine sh -c "exit 0" | 0 |
|   podman run --detach alpine sh -c "exit 1" | 0 |
|   podman run --detach alpine sh -c "exit 2" | 0 |
|   podman run --detach alpine sh -c "exit 3" | 0 |
|   podman run --detach alpine sh -c "exit 4" | 0 |
|        ... | ... |
|   podman run --detach alpine sh -c "exit 254" | 0 |
|   podman run --detach alpine sh -c "exit 255" | 0 |

(The lines for 5-253 were left out of the table for brevity)

__Example:__

```
$ podman run --rm --detach alpine sh -c "exit 1"
a42fec56914f777c7c79f5baad9ba169c3686177e92436181959e2fc1946eddd
$ echo $?
0
$
```

## Internal Podman error

In case Podman is not able to run the command, Podman uses a few exit status codes to indicate what went wrong.

|  internal Podman error | podman exit status |
| --                            |                --  |
| The error is with Podman itself | 125 |
| The container command cannot be invoked | 126 |
| The container command cannot be found | 127 |


__Example:__

Execute Podman with a command-line flag that does not exist:

```
$ podman run --foo alpine
Error: unknown flag: --foo
See 'podman run --help'
$ echo $?
125
```

__Example:__

Specify a container command that cannot be invoked. (_/etc_ is a directory and not an executable).
```
$ podman run alpine /etc
Error: crun: open executable: Operation not permitted: OCI permission denied
$ echo $?
126
```

__Example:__

Specify a container command that does not exist:
```
$ podman run --rm alpine asdfasdfsdf
Error: crun: executable file `asdfasdfsdf` not found in $PATH: No such file or directory: OCI runtime attempted to invoke a command that was not found
$ echo $?
127
```

Note that the podman exit status codes 125, 126, 127 can originate either from an internal Podman error or from the container command
having this exit status.

__Example:__

```
$ podman run alpine sh -c "exit 125"
$ echo $?
125
$ podman run alpine sh -c "exit 126"
$ echo $?
126
$ podman run alpine sh -c "exit 127"
$ echo $?
127
```

## Podman exit status in systemd service logs

When __podman run__ is executed in a systemd service and terminates with a non-zero exit status code, __systemd__ will
write the exit status code to the systemd journal log.
In addition to that __systemd__ appends a text annotation describing the specific exit status code.

| podman exit status | systemd log annotation |
|  --                |  --          |
| 1 | /FAILURE |
| 2 | /INVALIDARGUMENT |
| 3 | /NOTIMPLEMENTED |
| 4 | /NOPERMISSION |
| 5 | /NOTINSTALLED |
| 6 | /NOTCONFIGURED |
| 7 | /NOTRUNNING |
| 65 | /DATAERR |
| 66 | /NOINPUT |
| 67 | /NOUSER |
| 68 | /NOHOST |
| 69 | /UNAVAILABLE |
| 70 | /SOFTWARE |
| 71 | /OSERR |
| 72 | /OSFILE |
| 73 | /CANTCREAT |
| 74 | /IOERR |
| 75 | /TEMPFAIL |
| 76 | /PROTOCOL |
| 77 | /NOPERM |
| 78 | /CONFIG |

For all other non-zero error code numbers (8-64, 79-255) systemd appends the text string `/n/a` meaning no
annotation is available for the error status code.

Systemd services running Podman can be created by writing Quadlet files.
An alternative method (now deprecated) is the command [`podman generate systemd`](https://docs.podman.io/en/latest/markdown/podman-generate-systemd.1.html).

#### Example systemd service generated from a quadlet

The container command is configured to terminate with the exit status code `4`.

1. `sudo useradd test`
1. `sudo machinectl shell --uid test`
1. `mkdir -p ~/.config/containers/systemd`
1. Create the file _~/.config/containers/systemd/test@container_ with the contents
   ```
   [Container]
   Image=localhost/exit
   Exec=sh -c "exit 4"
   ```
1. Reload the systemd configuration
   ```
   systemctl --user daemon-reload
   ```
1. Start the service
   ```
   systemctl --user start test.service
   ```
1. Check the journal log
   ```
   journalctl -q --user --no-pager -xeu test.service | grep status=
   ```
   The command prints the output
   ```
   Apr 13 09:38:11 localhost.localdomain systemd[2877]: test.service: Main process exited, code=exited, status=4/NOPERMISSION
   ```
   The log message from systemd contains the annotation `/NOPERMISSION`

:note: The systemd annotation text can be misleading. For example `status=4/NOPERMISSION` seen in a journal log
merely indicates that the container command terminated with error status code 4. It is unclear whether the error was
really caused by a permission issue.

### Complete list of systemd exit status annotations

For completeness, here is a table showing all exit status codes and the text systemd writes to the systemd journal log.

| podman exit status | systemd service log output |
|  --                |  --                        |
| 0   | |
| 1   | Main process exited, code=exited, status=1/FAILURE |
| 2   | Main process exited, code=exited, status=2/INVALIDARGUMENT |
| 3   | Main process exited, code=exited, status=3/NOTIMPLEMENTED |
| 4   | Main process exited, code=exited, status=4/NOPERMISSION |
| 5   | Main process exited, code=exited, status=5/NOTINSTALLED |
| 6   | Main process exited, code=exited, status=6/NOTCONFIGURED |
| 7   | Main process exited, code=exited, status=7/NOTRUNNING |
| 8   | Main process exited, code=exited, status=8/n/a |
| 9   | Main process exited, code=exited, status=9/n/a |
| 10  | Main process exited, code=exited, status=10/n/a |
| 11  | Main process exited, code=exited, status=11/n/a |
| 12  | Main process exited, code=exited, status=12/n/a |
| 13  | Main process exited, code=exited, status=13/n/a |
| 14  | Main process exited, code=exited, status=14/n/a |
| 15  | Main process exited, code=exited, status=15/n/a |
| 16  | Main process exited, code=exited, status=16/n/a |
| 17  | Main process exited, code=exited, status=17/n/a |
| 18  | Main process exited, code=exited, status=18/n/a |
| 19  | Main process exited, code=exited, status=19/n/a |
| 20  | Main process exited, code=exited, status=20/n/a |
| 21  | Main process exited, code=exited, status=21/n/a |
| 22  | Main process exited, code=exited, status=22/n/a |
| 23  | Main process exited, code=exited, status=23/n/a |
| 24  | Main process exited, code=exited, status=24/n/a |
| 25  | Main process exited, code=exited, status=25/n/a |
| 26  | Main process exited, code=exited, status=26/n/a |
| 27  | Main process exited, code=exited, status=27/n/a |
| 28  | Main process exited, code=exited, status=28/n/a |
| 29  | Main process exited, code=exited, status=29/n/a |
| 30  | Main process exited, code=exited, status=30/n/a |
| 31  | Main process exited, code=exited, status=31/n/a |
| 32  | Main process exited, code=exited, status=32/n/a |
| 33  | Main process exited, code=exited, status=33/n/a |
| 34  | Main process exited, code=exited, status=34/n/a |
| 35  | Main process exited, code=exited, status=35/n/a |
| 36  | Main process exited, code=exited, status=36/n/a |
| 37  | Main process exited, code=exited, status=37/n/a |
| 38  | Main process exited, code=exited, status=38/n/a |
| 39  | Main process exited, code=exited, status=39/n/a |
| 40  | Main process exited, code=exited, status=40/n/a |
| 41  | Main process exited, code=exited, status=41/n/a |
| 42  | Main process exited, code=exited, status=42/n/a |
| 43  | Main process exited, code=exited, status=43/n/a |
| 44  | Main process exited, code=exited, status=44/n/a |
| 45  | Main process exited, code=exited, status=45/n/a |
| 46  | Main process exited, code=exited, status=46/n/a |
| 47  | Main process exited, code=exited, status=47/n/a |
| 48  | Main process exited, code=exited, status=48/n/a |
| 49  | Main process exited, code=exited, status=49/n/a |
| 50  | Main process exited, code=exited, status=50/n/a |
| 51  | Main process exited, code=exited, status=51/n/a |
| 52  | Main process exited, code=exited, status=52/n/a |
| 53  | Main process exited, code=exited, status=53/n/a |
| 54  | Main process exited, code=exited, status=54/n/a |
| 55  | Main process exited, code=exited, status=55/n/a |
| 56  | Main process exited, code=exited, status=56/n/a |
| 57  | Main process exited, code=exited, status=57/n/a |
| 58  | Main process exited, code=exited, status=58/n/a |
| 59  | Main process exited, code=exited, status=59/n/a |
| 60  | Main process exited, code=exited, status=60/n/a |
| 61  | Main process exited, code=exited, status=61/n/a |
| 62  | Main process exited, code=exited, status=62/n/a |
| 63  | Main process exited, code=exited, status=63/n/a |
| 64  | Main process exited, code=exited, status=64/n/a |
| 65  | Main process exited, code=exited, status=65/DATAERR |
| 66  | Main process exited, code=exited, status=66/NOINPUT |
| 67  | Main process exited, code=exited, status=67/NOUSER |
| 68  | Main process exited, code=exited, status=68/NOHOST |
| 69  | Main process exited, code=exited, status=69/UNAVAILABLE |
| 70  | Main process exited, code=exited, status=70/SOFTWARE |
| 71  | Main process exited, code=exited, status=71/OSERR |
| 72  | Main process exited, code=exited, status=72/OSFILE |
| 73  | Main process exited, code=exited, status=73/CANTCREAT |
| 74  | Main process exited, code=exited, status=74/IOERR |
| 75  | Main process exited, code=exited, status=75/TEMPFAIL |
| 76  | Main process exited, code=exited, status=76/PROTOCOL |
| 77  | Main process exited, code=exited, status=77/NOPERM |
| 78  | Main process exited, code=exited, status=78/CONFIG |
| 79  | Main process exited, code=exited, status=79/n/a |
| 80  | Main process exited, code=exited, status=80/n/a |
| 81  | Main process exited, code=exited, status=81/n/a |
| 82  | Main process exited, code=exited, status=82/n/a |
| 83  | Main process exited, code=exited, status=83/n/a |
| 84  | Main process exited, code=exited, status=84/n/a |
| 85  | Main process exited, code=exited, status=85/n/a |
| 86  | Main process exited, code=exited, status=86/n/a |
| 87  | Main process exited, code=exited, status=87/n/a |
| 88  | Main process exited, code=exited, status=88/n/a |
| 89  | Main process exited, code=exited, status=89/n/a |
| 90  | Main process exited, code=exited, status=90/n/a |
| 91  | Main process exited, code=exited, status=91/n/a |
| 92  | Main process exited, code=exited, status=92/n/a |
| 93  | Main process exited, code=exited, status=93/n/a |
| 94  | Main process exited, code=exited, status=94/n/a |
| 95  | Main process exited, code=exited, status=95/n/a |
| 96  | Main process exited, code=exited, status=96/n/a |
| 97  | Main process exited, code=exited, status=97/n/a |
| 98  | Main process exited, code=exited, status=98/n/a |
| 99  | Main process exited, code=exited, status=99/n/a |
| 100 | Main process exited, code=exited, status=100/n/a |
| 101 | Main process exited, code=exited, status=101/n/a |
| 102 | Main process exited, code=exited, status=102/n/a |
| 103 | Main process exited, code=exited, status=103/n/a |
| 104 | Main process exited, code=exited, status=104/n/a |
| 105 | Main process exited, code=exited, status=105/n/a |
| 106 | Main process exited, code=exited, status=106/n/a |
| 107 | Main process exited, code=exited, status=107/n/a |
| 108 | Main process exited, code=exited, status=108/n/a |
| 109 | Main process exited, code=exited, status=109/n/a |
| 110 | Main process exited, code=exited, status=110/n/a |
| 111 | Main process exited, code=exited, status=111/n/a |
| 112 | Main process exited, code=exited, status=112/n/a |
| 113 | Main process exited, code=exited, status=113/n/a |
| 114 | Main process exited, code=exited, status=114/n/a |
| 115 | Main process exited, code=exited, status=115/n/a |
| 116 | Main process exited, code=exited, status=116/n/a |
| 117 | Main process exited, code=exited, status=117/n/a |
| 118 | Main process exited, code=exited, status=118/n/a |
| 119 | Main process exited, code=exited, status=119/n/a |
| 120 | Main process exited, code=exited, status=120/n/a |
| 121 | Main process exited, code=exited, status=121/n/a |
| 122 | Main process exited, code=exited, status=122/n/a |
| 123 | Main process exited, code=exited, status=123/n/a |
| 124 | Main process exited, code=exited, status=124/n/a |
| 125 | Main process exited, code=exited, status=125/n/a |
| 126 | Main process exited, code=exited, status=126/n/a |
| 127 | Main process exited, code=exited, status=127/n/a |
| 128 | Main process exited, code=exited, status=128/n/a |
| 129 | Main process exited, code=exited, status=129/n/a |
| 130 | Main process exited, code=exited, status=130/n/a |
| 131 | Main process exited, code=exited, status=131/n/a |
| 132 | Main process exited, code=exited, status=132/n/a |
| 133 | Main process exited, code=exited, status=133/n/a |
| 134 | Main process exited, code=exited, status=134/n/a |
| 135 | Main process exited, code=exited, status=135/n/a |
| 136 | Main process exited, code=exited, status=136/n/a |
| 137 | Main process exited, code=exited, status=137/n/a |
| 138 | Main process exited, code=exited, status=138/n/a |
| 139 | Main process exited, code=exited, status=139/n/a |
| 140 | Main process exited, code=exited, status=140/n/a |
| 141 | Main process exited, code=exited, status=141/n/a |
| 142 | Main process exited, code=exited, status=142/n/a |
| 143 | Main process exited, code=exited, status=143/n/a |
| 144 | Main process exited, code=exited, status=144/n/a |
| 145 | Main process exited, code=exited, status=145/n/a |
| 146 | Main process exited, code=exited, status=146/n/a |
| 147 | Main process exited, code=exited, status=147/n/a |
| 148 | Main process exited, code=exited, status=148/n/a |
| 149 | Main process exited, code=exited, status=149/n/a |
| 150 | Main process exited, code=exited, status=150/n/a |
| 151 | Main process exited, code=exited, status=151/n/a |
| 152 | Main process exited, code=exited, status=152/n/a |
| 153 | Main process exited, code=exited, status=153/n/a |
| 154 | Main process exited, code=exited, status=154/n/a |
| 155 | Main process exited, code=exited, status=155/n/a |
| 156 | Main process exited, code=exited, status=156/n/a |
| 157 | Main process exited, code=exited, status=157/n/a |
| 158 | Main process exited, code=exited, status=158/n/a |
| 159 | Main process exited, code=exited, status=159/n/a |
| 160 | Main process exited, code=exited, status=160/n/a |
| 161 | Main process exited, code=exited, status=161/n/a |
| 162 | Main process exited, code=exited, status=162/n/a |
| 163 | Main process exited, code=exited, status=163/n/a |
| 164 | Main process exited, code=exited, status=164/n/a |
| 165 | Main process exited, code=exited, status=165/n/a |
| 166 | Main process exited, code=exited, status=166/n/a |
| 167 | Main process exited, code=exited, status=167/n/a |
| 168 | Main process exited, code=exited, status=168/n/a |
| 169 | Main process exited, code=exited, status=169/n/a |
| 170 | Main process exited, code=exited, status=170/n/a |
| 171 | Main process exited, code=exited, status=171/n/a |
| 172 | Main process exited, code=exited, status=172/n/a |
| 173 | Main process exited, code=exited, status=173/n/a |
| 174 | Main process exited, code=exited, status=174/n/a |
| 175 | Main process exited, code=exited, status=175/n/a |
| 176 | Main process exited, code=exited, status=176/n/a |
| 177 | Main process exited, code=exited, status=177/n/a |
| 178 | Main process exited, code=exited, status=178/n/a |
| 179 | Main process exited, code=exited, status=179/n/a |
| 180 | Main process exited, code=exited, status=180/n/a |
| 181 | Main process exited, code=exited, status=181/n/a |
| 182 | Main process exited, code=exited, status=182/n/a |
| 183 | Main process exited, code=exited, status=183/n/a |
| 184 | Main process exited, code=exited, status=184/n/a |
| 185 | Main process exited, code=exited, status=185/n/a |
| 186 | Main process exited, code=exited, status=186/n/a |
| 187 | Main process exited, code=exited, status=187/n/a |
| 188 | Main process exited, code=exited, status=188/n/a |
| 189 | Main process exited, code=exited, status=189/n/a |
| 190 | Main process exited, code=exited, status=190/n/a |
| 191 | Main process exited, code=exited, status=191/n/a |
| 192 | Main process exited, code=exited, status=192/n/a |
| 193 | Main process exited, code=exited, status=193/n/a |
| 194 | Main process exited, code=exited, status=194/n/a |
| 195 | Main process exited, code=exited, status=195/n/a |
| 196 | Main process exited, code=exited, status=196/n/a |
| 197 | Main process exited, code=exited, status=197/n/a |
| 198 | Main process exited, code=exited, status=198/n/a |
| 199 | Main process exited, code=exited, status=199/n/a |
| 200 | Main process exited, code=exited, status=200/n/a |
| 201 | Main process exited, code=exited, status=201/n/a |
| 202 | Main process exited, code=exited, status=202/n/a |
| 203 | Main process exited, code=exited, status=203/n/a |
| 204 | Main process exited, code=exited, status=204/n/a |
| 205 | Main process exited, code=exited, status=205/n/a |
| 206 | Main process exited, code=exited, status=206/n/a |
| 207 | Main process exited, code=exited, status=207/n/a |
| 208 | Main process exited, code=exited, status=208/n/a |
| 209 | Main process exited, code=exited, status=209/n/a |
| 210 | Main process exited, code=exited, status=210/n/a |
| 211 | Main process exited, code=exited, status=211/n/a |
| 212 | Main process exited, code=exited, status=212/n/a |
| 213 | Main process exited, code=exited, status=213/n/a |
| 214 | Main process exited, code=exited, status=214/n/a |
| 215 | Main process exited, code=exited, status=215/n/a |
| 216 | Main process exited, code=exited, status=216/n/a |
| 217 | Main process exited, code=exited, status=217/n/a |
| 218 | Main process exited, code=exited, status=218/n/a |
| 219 | Main process exited, code=exited, status=219/n/a |
| 220 | Main process exited, code=exited, status=220/n/a |
| 221 | Main process exited, code=exited, status=221/n/a |
| 222 | Main process exited, code=exited, status=222/n/a |
| 223 | Main process exited, code=exited, status=223/n/a |
| 224 | Main process exited, code=exited, status=224/n/a |
| 225 | Main process exited, code=exited, status=225/n/a |
| 226 | Main process exited, code=exited, status=226/n/a |
| 227 | Main process exited, code=exited, status=227/n/a |
| 228 | Main process exited, code=exited, status=228/n/a |
| 229 | Main process exited, code=exited, status=229/n/a |
| 230 | Main process exited, code=exited, status=230/n/a |
| 231 | Main process exited, code=exited, status=231/n/a |
| 232 | Main process exited, code=exited, status=232/n/a |
| 233 | Main process exited, code=exited, status=233/n/a |
| 234 | Main process exited, code=exited, status=234/n/a |
| 235 | Main process exited, code=exited, status=235/n/a |
| 236 | Main process exited, code=exited, status=236/n/a |
| 237 | Main process exited, code=exited, status=237/n/a |
| 238 | Main process exited, code=exited, status=238/n/a |
| 239 | Main process exited, code=exited, status=239/n/a |
| 240 | Main process exited, code=exited, status=240/n/a |
| 241 | Main process exited, code=exited, status=241/n/a |
| 242 | Main process exited, code=exited, status=242/n/a |
| 243 | Main process exited, code=exited, status=243/n/a |
| 244 | Main process exited, code=exited, status=244/n/a |
| 245 | Main process exited, code=exited, status=245/n/a |
| 246 | Main process exited, code=exited, status=246/n/a |
| 247 | Main process exited, code=exited, status=247/n/a |
| 248 | Main process exited, code=exited, status=248/n/a |
| 249 | Main process exited, code=exited, status=249/n/a |
| 250 | Main process exited, code=exited, status=250/n/a |
| 251 | Main process exited, code=exited, status=251/n/a |
| 252 | Main process exited, code=exited, status=252/n/a |
| 253 | Main process exited, code=exited, status=253/n/a |
| 254 | Main process exited, code=exited, status=254/n/a |
| 255 | Main process exited, code=exited, status=255/n/a |
