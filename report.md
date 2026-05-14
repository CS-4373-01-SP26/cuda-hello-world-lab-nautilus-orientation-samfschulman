Started out by installing `kubectl` and `kubelogin` from `snap`, since I'm on Ubuntu. Then I had to install the `oidc-login` plugin for `kubectl`, which required that I install the `krew` plugin first ([from here](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)) so that I could install other plugins. Then, and only then, did `kubectl get pods -n tandy-hpc` work. It opened a browser window to sign into authentik. I see a `hello-dev-pod`. `kubectl auth whoami` shows the following:

```
ATTRIBUTE   VALUE
Username    http://cilogon.org/serverE/users/494683
Groups      [oidcgroup:tandy-hpc oidcgroup:nautilus-user system:authenticated]
```

I guess my username is `494683`? Not really sure how else to read that. The link gets me a 404. Guess I'll just go with it. Got an error at first but that was just because the username wasn't properly quoted out, and kube was expecting a string. With that fixed, it proceeded smoothly from here.

## Questions

**Q1.** What GPU model was allocated to your pod? What is its compute
capability (the sm_XX number)? How many streaming multiprocessors (SMs)
and CUDA cores does it have? (Look up the spec sheet for the model name
shown by `nvidia-smi`.)

It was all Nvidia 1080-Ti. According to nvidia, this card has a compute capability of 6.1. As for streaming multiprocessors, it seems wikipedia is the only place that still has this particular level of spec information available for this old of a card, but it reports 28 SMs. Number of cuda cores seems to be 3584.

**Q2.** When you ran `./cuda_hello 10` multiple times, did the threads
always print their greetings in order (thread 0 first, then 1, 2, ...)?
Paste your loop output as evidence. Based on Pacheco §6.4–6.5, why
might the output order vary — or why might it always be the same?

The output order was the same every time, seen [here](run_10.txt). I think it's because with only 10 threads, we were entirely within a single warp, and threads within a warp run in lock-step.

**Q3.** The kernel is launched with `<<<1, thread_count>>>`. What does
the `1` represent? If you changed the launch to `<<<2, thread_count/2>>>`
for an even `thread_count`, what would change about the output? (Reason
from §6.6 — you do not need to modify and rerun the code.)

The 1 tells CUDA to create one block of threads. If we changed it as indicated, we would now have two separate blocks. The result would be that instead of seeing hello from threads 0-9, we would see 0-4, with each number appearing twice. The order would also be unpredictable, since now that we have two separate blocks, they could theoretically be assigned out to different SMs.

**Q4.** Experiment: what is the largest `thread_count` you can pass
before the program fails to produce output or crashes? What CUDA limit
does this correspond to? (See §6.7, Table 6.3 and the compute
capability of your GPU.)

1024 is the limit! It failed silently at 1025. This is the maximum number of threads per block for compute capability 6.1.

**Q5.** Why is `cudaDeviceSynchronize()` necessary in `main`? What
would happen if it were removed? (Reason from §6.5 — do not modify
the code to test this.)

Without that call, the CPU would fire off the jobs to the GPU and immediately proceed with its work. In this case, that means returning 0 and immediately exiting, which would then cause all the GPU threads to be killed. Not ideal.