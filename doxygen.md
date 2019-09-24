## doxygen

### install

```sh
sudo apt-get install doxygen graphviz
```

### config


```sh
doxygen -g
```

modify the "Doxyfile":

```
PROJECT_NAME = project
CALL_GRAPH = YES
HAVE_DOT = YES
RECURSIVE = YES
EXCLUDE_PATTERNS = */.git/*
EXCLUDE_PATTERNS += */docs/*
EXCLUDE_PATTERNS += */test/*
``` 

### generate

CD the code path:

```sh
doxygen ${Doxyfile path}
```

open html/index.html;
