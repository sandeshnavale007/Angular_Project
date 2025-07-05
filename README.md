@Configuration
@RequiredArgsConstructor
public class UserBatchJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final EntityManagerFactory entityManagerFactory;
    private final UserRepository userRepository;

    // === Step Scope Reader using findByIdIn + paging ===
    @Bean
    @StepScope
    public ItemReader<User> userReader(
            @Value("#{stepExecutionContext[minId]}") Long minId,
            @Value("#{stepExecutionContext[maxId]}") Long maxId
    ) {
        return new PaginatedUserReader(userRepository, minId, maxId);
    }

    // === Processor ===
    @Bean
    public ItemProcessor<User, User> userProcessor() {
        return user -> {
            if (user.getName() != null && !user.getName().isBlank()) {
                user.setStatus("PASS");
            } else {
                user.setStatus("FAIL");
            }
            return user;
        };
    }

    // === Writer ===
    @Bean
    @StepScope
    public ItemWriter<User> userWriter() {
        JpaItemWriter<User> writer = new JpaItemWriter<>();
        writer.setEntityManagerFactory(entityManagerFactory);
        return writer;
    }

    // === Worker Step ===
    @Bean
    public Step workerStep() {
        return new StepBuilder("workerStep", jobRepository)
                .<User, User>chunk(100, transactionManager)
                .reader(userReader(null, null))
                .processor(userProcessor())
                .writer(userWriter())
                .faultTolerant()
                .retryLimit(3)
                .retry(TransientDataAccessException.class)
                .skipLimit(10)
                .skip(DataIntegrityViolationException.class)
                .build();
    }

    // === Partitioner ===
    @Bean
    public Partitioner partitioner() {
        return gridSize -> {
            Long minId = userRepository.findMinId();
            Long maxId = userRepository.findMaxId();

            long range = (maxId - minId + 1) / gridSize;

            Map<String, ExecutionContext> partitions = new HashMap<>();
            long start = minId;
            long end = start + range - 1;

            for (int i = 0; i < gridSize; i++) {
                ExecutionContext context = new ExecutionContext();
                context.putLong("minId", start);
                context.putLong("maxId", Math.min(end, maxId));
                partitions.put("partition" + i, context);

                start = end + 1;
                end = start + range - 1;
            }

            return partitions;
        };
    }

    // === Master Step ===
    @Bean
    public Step masterStep() {
        return new StepBuilder("masterStep", jobRepository)
                .partitioner("workerStep", partitioner())
                .step(workerStep())
                .gridSize(4)
                .taskExecutor(new SimpleAsyncTaskExecutor("partition-"))
                .build();
    }

    // === Job ===
    @Bean
    public Job userStatusJob() {
        return new JobBuilder("userStatusJob", jobRepository)
                .start(masterStep())
                .build();
    }
}



@Bean
@StepScope
public JpaPagingItemReader<User> userReader(
        @Value("#{stepExecutionContext[minId]}") Long minId,
        @Value("#{stepExecutionContext[maxId]}") Long maxId,
        EntityManagerFactory entityManagerFactory) {

    JpaPagingItemReader<User> reader = new JpaPagingItemReader<>();
    reader.setEntityManagerFactory(entityManagerFactory);
    reader.setQueryString("""
        SELECT DISTINCT u FROM User u
        LEFT JOIN FETCH u.addresses
        WHERE u.id BETWEEN :minId AND :maxId
    """);
    reader.setParameterValues(Map.of("minId", minId, "maxId", maxId));
    reader.setPageSize(50);
    reader.setSaveState(false); // safe for partitioning
    return reader;
}




=====
@Component
public class ActiveStatusPartitioner implements Partitioner {

    @Autowired private MyEntityRepository repo;  // JPA repo

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        long min = repo.minIdByStatus(true);
        long max = repo.maxIdByStatus(true);
        long range = (max - min + 1) / gridSize;

        Map<String, ExecutionContext> partitions = new HashMap<>();
        long start = min, end = start + range - 1;

        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("startId", start);
            ctx.putLong("endId", i == gridSize - 1 ? max : end);
            partitions.put("partition" + i, ctx);
            start = end + 1;
            end = start + range - 1;
        }
        return partitions;
    }
}









======================
@Configuration
public class BatchConfig {

    private static final int GRID_SIZE = 4, CHUNK_SIZE = 100;

    @Bean
    public Step workerStep(JobRepository jobRepo,
                           PlatformTransactionManager txMgr,
                           JpaPagingItemReader<MyEntity> reader,
                           ItemProcessor<MyEntity,MyEntity> processor,
                           ItemWriter<MyEntity> writer,
                           SkipListener<MyEntity,MyEntity> skipListener) {
        return new StepBuilder("workerStep", jobRepo)
            .<MyEntity,MyEntity>chunk(CHUNK_SIZE, txMgr)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
               .retryLimit(3)
               .retry(DeadlockLoserDataAccessException.class)
               .skipLimit(10)
               .skip(Exception.class)
               .noSkip(NullPointerException.class)
               .listener(skipListener)
            .build();
    }

    @Bean
    public Job partitionedJob(JobRepository jobRepo,
                              TaskExecutor taskExecutor,
                              JobLifecycleListener listener,
                              ActiveStatusPartitioner partitioner,
                              Step workerStep) {
        Step master = new StepBuilder("masterStep", jobRepo)
            .partitioner(workerStep.getName(), partitioner)
            .step(workerStep)
            .taskExecutor(taskExecutor)
            .gridSize(GRID_SIZE)
            .build();

        return new JobBuilder("partitionJob", jobRepo)
            .listener(listener)
            .start(master)
            .build();
    }

    @Bean @StepScope
    public JpaPagingItemReader<MyEntity> reader(
        EntityManagerFactory emf,
        @Value("#{stepExecutionContext['startId']}") long start,
        @Value("#{stepExecutionContext['endId']}") long end
    ) {
        return new JpaPagingItemReaderBuilder<MyEntity>()
            .name("jpaReader")
            .entityManagerFactory(emf)
            .queryString("""
               SELECT e FROM MyEntity e
               WHERE e.active = true AND e.id BETWEEN :start AND :end
            """)
            .parameterValues(Map.of("start", start, "end", end))
            .pageSize(CHUNK_SIZE)
            .build();
    }

    @Bean
    public ItemProcessor<MyEntity,MyEntity> processor() {
        return e -> {
            boolean ok = e.getSomeField() != null;
            e.setStatus(ok ? "PASS" : "FAIL");
            return e;
        };
    }

    @Bean @StepScope
    public ItemWriter<MyEntity> writer(EntityManagerFactory emf) {
        var delegate = new JpaItemWriter<MyEntity>();
        delegate.setEntityManagerFactory(emf);
        var sync = new SynchronizedItemStreamWriter<MyEntity>();
        sync.setDelegate(delegate);
        return sync;
    }

    @Bean
    public SkipListener<MyEntity,MyEntity> skipListener() {
        return new SkipListener<>() {
            public void onSkipInRead(Throwable t)    { System.err.println("Skip read: " + t); }
            public void onSkipInProcess(MyEntity i,Throwable t) { System.err.println("Skip proc: " + i); }
            public void onSkipInWrite(MyEntity i,Throwable t) { System.err.println("Skip write: " + i); }
        };
    }

    @Bean
    public TaskExecutor taskExecutor() {
        var te = new ThreadPoolTaskExecutor();
        te.setCorePoolSize(GRID_SIZE);
        te.setMaxPoolSize(GRID_SIZE);
        te.setThreadNamePrefix("part-");
        te.initialize();
        return te;
    }
}










@Component
public class JobLifecycleListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("🚀 Job " + jobExecution.getJobInstance().getJobName() + " starting at " +
                           jobExecution.getStartTime());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        BatchStatus status = jobExecution.getStatus();
        if (status == BatchStatus.COMPLETED) {
            System.out.println("✅ Job completed successfully at " + jobExecution.getEndTime());
        } else {
            System.out.println("❌ Job ended with status: " + status +
                               " and exit status: " + jobExecution.getExitStatus());
        }
    }
}




@Component
public class JobLifecycleListener implements JobExecutionListener {
    @Override
    public void beforeJob(JobExecution je) {
        System.out.println("🚀 Starting job: " + je.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution je) {
        System.out.println("🏁 Job ended with status: " + je.getStatus());
    }
}
