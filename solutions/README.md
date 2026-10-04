# Reference Solutions

Live-delivery solutions are **not** kept on `main`. Each class gets its own
branch (`agy_aug2026`, `agy_sep2026`, `agy_oct2026`, …) where the instructor
commits what was built in front of the room — for example the Node.js task
manager from Lab 1 and the bookstore-api hardening from Labs 2–4.

```bash
git branch -r | grep agy_      # list delivery branches
git checkout agy_sep2026       # browse one
```

`main` always holds the student baseline so the labs have something to build.
