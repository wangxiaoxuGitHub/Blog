`java.lang.invoke.SerializedLambda`

`java.util.concurrent.locks.ReentrantLock#lock`

`java.util.concurrent.atomic.AtomicReferenceArray`

`java.lang.ref.WeakReference`

@Bean
public SessionFactory sessionFactory(@Qualifier("entityManagerFactory") EntityManagerFactory emf) {
    return emf.unwrap(SessionFactory.class);
}


curl -sL https://rpm.nodesource.com/setup_10.x | bash -

sudo yum remove -y nodejs npm

sudo yum install -y nodejs

