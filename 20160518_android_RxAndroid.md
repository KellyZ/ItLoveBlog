## 概念

RxJava主要由以下几方面组成：

1. Observable（被订阅者）、OnSubscribe（可以称为回调）和subscribe（订阅者）；
2. Scheduler （用于异步规则）：subscribeOn、observeOn；
3. 变换：lift、map、flatMap、throttleFirst、compose

Observable相关：（Observable和Observable）

    class Observable<T>
        static <T> Observable<T> create(OnSubscribe<T> f)
        final <R> Observable<R> flatMap(Func1<? super T, ? extends Observable<? extends R>> func)
        final Observable<T> doOnSubscribe(final Action0 subscribe)
        Subscription subscribe(final Observer<? super T> observer)

    interface OnSubscribe<T> extends Action1<Subscriber<? super T>>
    interface Action1<T> extends Action
    interface Action extends Function

Subscribe相关：

    interface Subscription
        void unsubscribe();
        boolean isUnsubscribed();
    
    abstract class Subscriber<T> implements Observer<T>, Subscription

## 参考

1. http://gank.io/post/560e15be2dca930e00da1083#toc_1