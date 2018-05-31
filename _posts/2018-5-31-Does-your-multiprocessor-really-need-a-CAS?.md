
Your processor most likely has a compare-and-swap (CAS) instruction that can be used for synchronization when programming a multi-threaded application. You can think of a CAS instruction as one that takes three arguments:

```
boolean CAS(int *p, int v1, int v2) {
	if (*p == v1) {
        *p = v2;
        return true;    
    }
    return false;
}
```