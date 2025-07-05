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
@EnableBatchProcessing
public class BatchConfig {
    private static final int GRID_SIZE = 4, CHUNK_SIZE = 100;

    private final JobBuilderFactory jobs;
    private final StepBuilderFactory steps;
    private final EntityManagerFactory emf;
    private final MyEntityRepository repo;
    private final ActiveStatusPartitioner partitioner;

    public BatchConfig(JobBuilderFactory jobs,
                       StepBuilderFactory steps,
                       EntityManagerFactory emf,
                       MyEntityRepository repo,
                       ActiveStatusPartitioner partitioner) {
        this.jobs = jobs;
        this.steps = steps;
        this.emf = emf;
        this.repo = repo;
        this.partitioner = partitioner;
    }

    @Bean @StepScope
    public JpaPagingItemReader<MyEntity> reader(
            @Value("#{stepExecutionContext['startId']}") long startId,
            @Value("#{stepExecutionContext['endId']}") long endId) {
        return new JpaPagingItemReaderBuilder<MyEntity>()
                .name("jpaReader")
                .entityManagerFactory(emf)
                .queryString("SELECT e FROM MyEntity e WHERE e.active = true AND e.id BETWEEN :start AND :end")
                .parameterValues(Map.of("start", startId, "end", endId))
                .pageSize(CHUNK_SIZE)
                .build();
    }

    @Bean
    public ItemProcessor<MyEntity, MyEntity> processor() {
        return entity -> {
            boolean ok = entity.getSomeField() != null;
            entity.setStatus(ok ? "PASS" : "FAIL");
            return entity;
        };
    }

    @Bean @StepScope
    public ItemWriter<MyEntity> writer() {
        JpaItemWriter<MyEntity> delegate = new JpaItemWriter<>();
        delegate.setEntityManagerFactory(emf);
        SynchronizedItemStreamWriter<MyEntity> sync = new SynchronizedItemStreamWriter<>();
        sync.setDelegate(delegate);
        return sync;
    }

    @Bean
    public SkipListener<MyEntity, MyEntity> skipListener() {
        return new SkipListener<>() {
            @Override public void onSkipInRead(Throwable t) { System.err.println("Skipped read: " + t); }
            @Override public void onSkipInProcess(MyEntity item, Throwable t) { System.err.println("Skipped process: " + item + ", " + t); }
            @Override public void onSkipInWrite(MyEntity item, Throwable t) { System.err.println("Skipped write: " + item + ", " + t); }
        };
    }

    @Bean
    public Step workerStep(PlatformTransactionManager txManager) {
        return steps.get("workerStep")
            .<MyEntity, MyEntity>chunk(CHUNK_SIZE)
            .reader(reader(0,0))
            .processor(processor())
            .writer(writer())
            .faultTolerant()
                .retryLimit(3)
                .retry(DeadlockLoserDataAccessException.class)
                .skipLimit(10)
                .skip(Exception.class)
                .noSkip(NullPointerException.class)
                .listener(skipListener())
            .transactionManager(txManager)
            .build();
    }

    @Bean
    public Step masterStep(TaskExecutor executor, PlatformTransactionManager txManager) {
        return steps.get("masterStep")
            .partitioner(workerStep(txManager).getName(), partitioner)
            .step(workerStep(txManager))
            .taskExecutor(executor)
            .gridSize(GRID_SIZE)
            .build();
    }

    @Bean
    public Job partitionedJob(TaskExecutor executor,
                              PlatformTransactionManager txManager,
                              JobLifecycleListener jobListener) {
        return jobs.get("partitionedJob")
            .listener(jobListener) // integrates beforeJob/afterJob
            .start(masterStep(executor, txManager))
            .build();
    }

    @Bean
    public TaskExecutor taskExecutor() {
        var exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(GRID_SIZE);
        exec.setMaxPoolSize(GRID_SIZE);
        exec.setThreadNamePrefix("partition-");
        exec.initialize();
        return exec;
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
