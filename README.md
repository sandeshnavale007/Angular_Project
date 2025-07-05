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

