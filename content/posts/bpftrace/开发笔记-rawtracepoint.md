rawtracepointbtf:net_dev_xmit { 
    printf("%d\n", args->arg1->sk);
}