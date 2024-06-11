---
cascade:
- _target:
    environment: development
  build:
    list: always
- _target:
    environment: '{staging,production}'
  build:
    list: never
---
