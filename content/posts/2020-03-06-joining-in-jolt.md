---
title:  Joining in Jolt
author: Matthew
date: 2020-03-06
---

I stumbled across a question on [stackoverflow][1] that was asking how to perform lookup based on ids in a JSON document. After some initial thought and trial and error (because thats the only way you can develop [JOLT][jolt]), I came up with a solution that is repeatable allowing you to perform multiple joins.

Firstly lets look at the input and outputs:

## Input JSON

Contains a data array and relationships array.

```json
{
  "data": [
    {
      "id": "1112245810",
      "title": "Introduction to JavaScript Object Notation",
      "author": "54256",
      "publisher": "57756",
      "edition": "first",
      "published": "2012"
    },
    {
      "id": "1156464683",
      "title": "JSON at Work",
      "author": "15467",
      "publisher": "57756",
      "edition": "second",
      "published": "2014"
    },
    {
      "id": "1004467968",
      "title": "A Tiny Bit Mortal",
      "author": "54256",
      "publisher": "56465",
      "edition": "first",
      "published": "2018"
    }
  ],
  "relationships": [
    {
      "id": "54256",
      "type": "author",
      "attributes": {
        "first-name": "Lindsay",
        "last-name": "Bassett",
        "city": "Michigan"
      }
    },
    {
      "id": "15467",
      "type": "author",
      "attributes": {
        "first-name": "Tom",
        "last-name": "Marrs",
        "city": "Cologne"
      }
    },
    {
      "id": "57756",
      "type": "publisher",
      "attributes": {
        "name": "O'REILLY",
        "established": "1984"
      }
    },
     {
      "id": "56465",
      "type": "publisher",
      "attributes": {
        "name": "APRESS",
        "established": "1979"
      }
    }
  ]
}
```

## Output JSON

Which is essentially taking copying the relationships based on type and id.

```json
{
  "data": [
    {
      "id": "1112245810",
      "title": "Introduction to JavaScript Object Notation",
      "author": {
        "first-name": "Lindsay",
        "last-name": "Bassett",
        "city": "Michigan"
      },
      "publisher": {
        "name": "O'REILLY",
        "established": "1984"
      },
      "edition": "first",
      "published": "2012"
    },
    {
      "id": "1156464683",
      "title": "JSON at Work",
      "author": {
        "first-name": "Tom",
        "last-name": "Marrs",
        "city": "Cologne"
      },
      "publisher": {
        "name": "O'REILLY",
        "established": "1984"
      },
      "edition": "second",
      "published": "2014"
    },
    {
      "id": "1004467968",
      "title": "A Tiny Bit Mortal",
      "author": {
        "first-name": "Lindsay",
        "last-name": "Bassett",
        "city": "Michigan"
      },
      "publisher": {
        "name": "APRESS",
        "established": "1979"
      },
      "edition": "first",
      "published": "2018"
    }
  ]
}
```

## Transformation

In order to achieve this i wrote a mutli stage shift, the main principle behind it is keying JSON objects on the ID, then copying the object across based on the ID, and shifting back out of it.

This produces the desired result (and possibly could be refactored further).

The main idea around it is setting the JSON to be keyed by something it can then be "joined" by. See the outputs of each shift and brief explanation below.

The shifts work as follows:

  1. Shift `relationships` by there `type` and `id`s
  2. Shift `data` by `type` `id`
  3. Shift `relationships` for a `type` into `data` by `type` `id`s
  4. Shift `type` into each `data` element key by `data` `id`
  5. Repeat 2-4 as much as needed
  6. Shift to final output

### Shift `relationships` by there `type` and `id`s

```json
{
  "operation": "shift",
  "spec": {
    "*": "&",
    "relationships": {
      "*": {
        "type": {
          "*": {
            "@(2,attributes)": "@2.@(3,id)"
          }
        }
      }
    }
  }
}
```

```json
{
  "author" : {
    "54256" : {
      "first-name" : "Lindsay",
      "last-name" : "Bassett",
      "city" : "Michigan"
    },
    "15467" : {
      "first-name" : "Tom",
      "last-name" : "Marrs",
      "city" : "Cologne"
    }
  },
  "publisher" : {
    "57756" : {
      "name" : "O'REILLY",
      "established" : "1984"
    },
    "56465" : {
      "name" : "APRESS",
      "established" : "1979"
    }
  },
  "data" : [ {
    "id" : "1112245810",
    "title" : "Introduction to JavaScript Object Notation",
    "author" : "54256",
    "publisher" : "57756",
    "edition" : "first",
    "published" : "2012"
  }, {
    "id" : "1156464683",
    "title" : "JSON at Work",
    "author" : "15467",
    "publisher" : "57756",
    "edition" : "second",
    "published" : "2014"
  }, {
    "id" : "1004467968",
    "title" : "A Tiny Bit Mortal",
    "author" : "54256",
    "publisher" : "56465",
    "edition" : "first",
    "published" : "2018"
  } ]
}

```

### Shift `data` by `type` `id`

```json
{
  "operation": "shift",
  "spec": {
    "*": "&",
    "data": {
      "*": {
        "author": {
          "*": {
            "@(2)": "data.@2.data.[]"
          }
        }
      }
    }
  }
}
```

```json
{
  "data" : {
    "54256" : {
      "data" : [ {
        "id" : "1112245810",
        "title" : "Introduction to JavaScript Object Notation",
        "author" : "54256",
        "publisher" : "57756",
        "edition" : "first",
        "published" : "2012"
      }, {
        "id" : "1004467968",
        "title" : "A Tiny Bit Mortal",
        "author" : "54256",
        "publisher" : "56465",
        "edition" : "first",
        "published" : "2018"
      } ]
    },
    "15467" : {
      "data" : [ {
        "id" : "1156464683",
        "title" : "JSON at Work",
        "author" : "15467",
        "publisher" : "57756",
        "edition" : "second",
        "published" : "2014"
      } ]
    }
  },
  "author" : {
    "54256" : {
      "first-name" : "Lindsay",
      "last-name" : "Bassett",
      "city" : "Michigan"
    },
    "15467" : {
      "first-name" : "Tom",
      "last-name" : "Marrs",
      "city" : "Cologne"
    }
  },
  "publisher" : {
    "57756" : {
      "name" : "O'REILLY",
      "established" : "1984"
    },
    "56465" : {
      "name" : "APRESS",
      "established" : "1979"
    }
  }
}
```

### Shift `relationships` for a `type` into `data` by `type` `id`s

```json
{
  "operation": "shift",
  "spec": {
    "*": "&",
    "author": {
      "*": {
        "@": "data.&.author"
      }
    }
  }
}
```

```json
{
  "data" : {
    "54256" : {
      "data" : [ {
        "id" : "1112245810",
        "title" : "Introduction to JavaScript Object Notation",
        "author" : "54256",
        "publisher" : "57756",
        "edition" : "first",
        "published" : "2012"
      }, {
        "id" : "1004467968",
        "title" : "A Tiny Bit Mortal",
        "author" : "54256",
        "publisher" : "56465",
        "edition" : "first",
        "published" : "2018"
      } ],
      "author" : {
        "first-name" : "Lindsay",
        "last-name" : "Bassett",
        "city" : "Michigan"
      }
    },
    "15467" : {
      "data" : [ {
        "id" : "1156464683",
        "title" : "JSON at Work",
        "author" : "15467",
        "publisher" : "57756",
        "edition" : "second",
        "published" : "2014"
      } ],
      "author" : {
        "first-name" : "Tom",
        "last-name" : "Marrs",
        "city" : "Cologne"
      }
    }
  },
  "publisher" : {
    "57756" : {
      "name" : "O'REILLY",
      "established" : "1984"
    },
    "56465" : {
      "name" : "APRESS",
      "established" : "1979"
    }
  }
}
```

### Shift `type` into each `data` element key by `data` `id`

```json
{
  "operation": "shift",
  "spec": {
    "*": "&",
    "data": {
      "*": {
        "data": {
          "*": {
            "@(id)": "data.@(id).id",
            "@(title)": "data.@(id).title",
            "@(2,author)": "data.@(id).author",
            "@(publisher)": "data.@(id).publisher",
            "@(edition)": "data.@(id).edition",
            "@(published)": "data.@(id).published"
          }
        }
      }
    }
  }
}
```

```json
{
  "data" : {
    "1112245810" : {
      "id" : "1112245810",
      "title" : "Introduction to JavaScript Object Notation",
      "author" : {
        "first-name" : "Lindsay",
        "last-name" : "Bassett",
        "city" : "Michigan"
      },
      "publisher" : "57756",
      "edition" : "first",
      "published" : "2012"
    },
    "1004467968" : {
      "id" : "1004467968",
      "title" : "A Tiny Bit Mortal",
      "author" : {
        "first-name" : "Lindsay",
        "last-name" : "Bassett",
        "city" : "Michigan"
      },
      "publisher" : "56465",
      "edition" : "first",
      "published" : "2018"
    },
    "1156464683" : {
      "id" : "1156464683",
      "title" : "JSON at Work",
      "author" : {
        "first-name" : "Tom",
        "last-name" : "Marrs",
        "city" : "Cologne"
      },
      "publisher" : "57756",
      "edition" : "second",
      "published" : "2014"
    }
  },
  "publisher" : {
    "57756" : {
      "name" : "O'REILLY",
      "established" : "1984"
    },
    "56465" : {
      "name" : "APRESS",
      "established" : "1979"
    }
  }
}

```

### Repeat 2-4 as much as needed

_The same as above but for publisher._

```json
[
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "publisher": {
            "*": {
              "@(2)": "data.@2.data.[]"
            }
          }
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "publisher": {
        "*": {
          "@": "data.&.publisher"
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "data": {
            "*": {
              "@(id)": "data.@(id).id",
              "@(title)": "data.@(id).title",
              "@(author)": "data.@(id).author",
              "@(2,publisher)": "data.@(id).publisher",
              "@(edition)": "data.@(id).edition",
              "@(published)": "data.@(id).published"
            }
          }
        }
      }
    }
  }
]
```

```json
{
  "data" : {
    "1112245810" : {
      "id" : "1112245810",
      "title" : "Introduction to JavaScript Object Notation",
      "author" : {
        "first-name" : "Lindsay",
        "last-name" : "Bassett",
        "city" : "Michigan"
      },
      "publisher" : {
        "name" : "O'REILLY",
        "established" : "1984"
      },
      "edition" : "first",
      "published" : "2012"
    },
    "1156464683" : {
      "id" : "1156464683",
      "title" : "JSON at Work",
      "author" : {
        "first-name" : "Tom",
        "last-name" : "Marrs",
        "city" : "Cologne"
      },
      "publisher" : {
        "name" : "O'REILLY",
        "established" : "1984"
      },
      "edition" : "second",
      "published" : "2014"
    },
    "1004467968" : {
      "id" : "1004467968",
      "title" : "A Tiny Bit Mortal",
      "author" : {
        "first-name" : "Lindsay",
        "last-name" : "Bassett",
        "city" : "Michigan"
      },
      "publisher" : {
        "name" : "APRESS",
        "established" : "1979"
      },
      "edition" : "first",
      "published" : "2018"
    }
  }
}
```

### Shift to final output

Remove data id as key:

```json
{
  "operation": "shift",
  "spec": {
    "*": "&",
    "data": {
      "*": {
        "@": "data.[]"
      }
    }
  }
}
```

```json
{
  "data" : [ {
    "id" : "1112245810",
    "title" : "Introduction to JavaScript Object Notation",
    "author" : {
      "first-name" : "Lindsay",
      "last-name" : "Bassett",
      "city" : "Michigan"
    },
    "publisher" : "57756",
    "edition" : "first",
    "published" : "2012"
  }, {
    "id" : "1004467968",
    "title" : "A Tiny Bit Mortal",
    "author" : {
      "first-name" : "Lindsay",
      "last-name" : "Bassett",
      "city" : "Michigan"
    },
    "publisher" : "56465",
    "edition" : "first",
    "published" : "2018"
  }, {
    "id" : "1156464683",
    "title" : "JSON at Work",
    "author" : {
      "first-name" : "Tom",
      "last-name" : "Marrs",
      "city" : "Cologne"
    },
    "publisher" : "57756",
    "edition" : "second",
    "published" : "2014"
  } ],
  "publisher" : {
    "57756" : {
      "name" : "O'REILLY",
      "established" : "1984"
    },
    "56465" : {
      "name" : "APRESS",
      "established" : "1979"
    }
  }
}
```

## Conclusion

A repeatable pattern that is airing on the side of "if all you have is a hammer, everything looks like a nail".

### Full Spec

```json
[
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "relationships": {
        "*": {
          "type": {
            "*": {
              "@(2,attributes)": "@2.@(3,id)"
            }
          }
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "author": {
            "*": {
              "@(2)": "data.@2.data.[]"
            }
          }
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "author": {
        "*": {
          "@": "data.&.author"
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "data": {
            "*": {
              "@(id)": "data.@(id).id",
              "@(title)": "data.@(id).title",
              "@(2,author)": "data.@(id).author",
              "@(publisher)": "data.@(id).publisher",
              "@(edition)": "data.@(id).edition",
              "@(published)": "data.@(id).published"
            }
          }
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "publisher": {
            "*": {
              "@(2)": "data.@2.data.[]"
            }
          }
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "publisher": {
        "*": {
          "@": "data.&.publisher"
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "data": {
            "*": {
              "@(id)": "data.@(id).id",
              "@(title)": "data.@(id).title",
              "@(author)": "data.@(id).author",
              "@(2,publisher)": "data.@(id).publisher",
              "@(edition)": "data.@(id).edition",
              "@(published)": "data.@(id).published"
            }
          }
        }
      }
    }
  },
  {
    "operation": "shift",
    "spec": {
      "*": "&",
      "data": {
        "*": {
          "@": "data.[]"
        }
      }
    }
  }
]
```

[1]:https://stackoverflow.com/a/60565994/1600422
[jolt]:https://github.com/bazaarvoice/jolt
