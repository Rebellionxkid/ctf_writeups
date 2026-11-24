# CTF writeups

My writeups for CTF challenges. I mostly write them for myself, so I don't have to solve the same thing twice six months later. If they help someone else, even better.

I play mostly web, crypto and forensics, with some pwn here and there. Reverse engineering is still at the "open it in IDA and hope" stage.

## Layout

Writeups are grouped by event. Inside there is one folder per challenge, named after the challenge. The category is written at the top of each challenge's README.

```
.
├── <event>/
│   ├── <challenge>/
│   │   ├── README.md      # the writeup itself, category is stated at the top
│   │   ├── solve.py       # only if there is a script
│   │   └── files/         # challenge files, only if I have them
│   └── <challenge>/
│       └── README.md
└── README.md
```

Not every challenge has a script or attached files. Some are solved by hand with a couple of tools, and then the writeup is just text and screenshots.

Each writeup tries to cover what the challenge gave me, how I got to the solution (dead ends included, they are usually the useful part) and the flag.

## Notes

Flags are left in because the events are over. Challenge files are only included when the organizers allow it and they are not huge. If something is missing, the writeup links to the original source where possible.

Scripts were written during the game, so don't expect clean code. They work though.

## Tools I use most

- Burp Suite, Postman and ffuf for web
- Python (pwntools, pycryptodome, requests)
- CyberChef when I'm too lazy to write a script
- Wireshark and Autopsy for forensics
- IDA Pro and Ghidra for reversing
- gdb with pwndbg for pwn

## Mistakes and questions

If you spot an error or know a simpler solution, open an issue or a pull request. I'm still learning, so I might take a while to answer.

## License

Feel free to use and retell the writeups, just link back to this repo. Code is under MIT.
