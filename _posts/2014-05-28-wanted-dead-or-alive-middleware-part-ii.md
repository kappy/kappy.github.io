---
published: true
title: 'Wanted dead or alive: Middleware (part II)'
layout: post
---

This is a series of blog posts that focuses on faulty middleware services invocations, and how can we reduce the impact of those faults.

[Part I](http://www.kspace.pt/posts/wanted-dead-or-alive-middleware-part-i)

Part II (this)

[Part III](http://www.kspace.pt/posts/wanted-dead-or-alive-middleware-part-iii)

The full source code can be located [here](https://github.com/kappy/FaultyMiddleware)

## Try Again, and Again, and Again...

If the error is in fact transient, we can avoid it if we retry the invocation.
The other way around, if the erros is caused by missdata that we send erroneously, then retrying won't do us any good. This is where we distinguish application errors or comunication errors. For simplicity sake we'll just consider the later one.

Ok, so let's get the party started. We need to repeate the service invocation if we got any exception. One way to do it, is to code the repeate logic on the client implementation. But what if we get more services? What if we get more actions to call? No... This is not the way to go.
What if we could put a transparent piece between the code that invokes the client and the client itself? Well, we actually can. It's called an **interceptor**!

To use an interceptor, first let's call a friend of mine. **Dependency Injection**.
For the sake of lenght of this post, I won't detail what DI is. Just Google it.
Also, for this example, I'll be using NInject and NInject.Extensions.Interception.LinFu packages for handling DI and Interception, but you can easly switch to the DI container of your own choosing. The important here is how the pieces work togheter, not what glues them.

##Show me the code already!

Ok, ok.. Let's refactor our NaiveClient, and extract interfaces on the public methods. Then we bind the interfaces and the implementations. Oh wait, did I said interfaces?
Right, for the interception work the best, we need to work with interfaces and not the concrete types themselfs. But that is already your common practice, right?

So, here's the glue:

    public class Module : NinjectModule
    {
        public override void Load()
        {
            Kernel.Bind<StatsCounter>().ToConstant(new StatsCounter());
            Kernel.Bind<INaiveClient>().To<NaiveClient>().Intercept().With<RetryInterceptor>();
        }
    }
    
What we're telling here is that `INaiveClient` is served by the type `NaiveClient` but has a `RetryInterceptor` between.
Here's the retry:

    public class RetryInterceptor : IInterceptor
    {
        private readonly StatsCounter _counter;
        private const int Tries = 3;

        public RetryInterceptor(StatsCounter counter)
        {
            _counter = counter;
        }

        public void Intercept(IInvocation invocation)
        {
            var tryNumber = 0;
            do
            {
                try
                {
                    invocation.Proceed();
                    return;
                }
                catch (Exception ex)
                {
                    tryNumber++;
                    if (tryNumber == Tries)
                    {
                        throw;
                    }
                }
            } while (true); 
        }
    }

In simple terms, it's invocating the `invocation.Proceed()` until it succeeds or we reach the maximum tries.
The `invocation.Proceed()` is the magic here. From the docs:

> Continues the invocation, either by invoking the next interceptor in the chain, or if there are no more interceptors, calling the target method.

We need also change our test program, to use the magic DI.

    class Program
    {
        const int TimesToInvoke = 1000;

        static void Main(string[] args)
        {
            var kernel = new StandardKernel();
            kernel.Load(new Module());

            var counter = kernel.Get<StatsCounter>();
            var client = kernel.Get<INaiveClient>();
            counter.Stopwatch.Start();
            for (var i = 0; i < TimesToInvoke; i++)
            {
                try
                {
                    client.GetMyDate(DateTime.Today.AddDays(i % 30));
                    counter.TotalSuccess++;
                }
                catch (Exception ex)
                {
                    counter.TotalError++;
                }
            }
            counter.Stopwatch.Stop();

            counter.PrintStats();
            Console.WriteLine("Press any key to exit");
            Console.ReadKey();
        }
    }
    
Take notice that we only work with `INaiveClient`. We don’t have any direct reference to the `RetryInterceptor`. The for cycle is the same, no changes what so ever.
![ClientImprovedStats](http://www.kspace.pt/images/blog/interceptor_II_zpssj2ud9ya.PNG)
This time we completed the 1000 invocations in 50 seconds, but we had only 28 failures. That's 2,8% error rate (again, trust my math). It's a really nicer number than 32,1% error rate from our last example, and we "didn’t change anything" on our client (only extracted an interface out of it, that should already be a practice). We just inserted a middle piece that extends the default behavior. The higher time can be easily justified by retries that were made. More 417 invocations were made to accomplish those retries.
Note also the Execution Fail number is a bit higher. That's also because we're doing more invocations.

##Does it get better than that?
This is a great improvement. We traded of some performance to gain some resilience. But can we improve this better, and of course, without changing any implementation?
Of course we can.
On my next post, I'll add some cache layer, again by using interception, and not changing anything on the client and the retries components.


[Part I](http://www.kspace.pt/posts/wanted-dead-or-alive-middleware-part-i/)

Part II (this)

[Part III](http://www.kspace.pt/posts/wanted-dead-or-alive-middleware-part-iii/)

The full source code can be located [here](https://github.com/kappy/FaultyMiddleware)
