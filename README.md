# Spring-Integration-Syslog
SpringIntegration을 활용하여 syslog 수집

- 주요 업무: Syslog 모듈 개발 및 배포
- 기술 스택 : Java, Springboot, Angular.js, Mybatis, PostgreSQL, Jenkins, Eureaka
- 개발 환경 : IntelliJ, Github, JIRA confluence, postman, swagger, DBeaver
- 업무 기간 : 2022.02 ~ 2022.03
- 개발 인원 : 3인
- 상세 내용 :
a. Spring Integration + Spring batch를 활용하여 장비 노드의 Log 메세지를 가져올 수 있는 Syslog 모듈을 개발
b. 고객사에 배포하여 테스트 진행

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
@EnableIntegration
@Slf4j
public class SyslogConfig extends DefaultBatchConfigurer {

    //private Job job; //순환참조 됨
    private final JobLauncher jobLauncher;
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final SyslogInsertMapper syslogInsertMapper;
    private final SyslogService syslogService;

    @Bean(name = PollerMetadata.DEFAULT_POLLER)
    public PollerMetadata defaultPoller() {
        PollerMetadata pollerMetadata = new PollerMetadata();
        pollerMetadata.setTrigger(new PeriodicTrigger(10));
        return pollerMetadata;
    }


    @ServiceActivator(inputChannel = "tailChannel", outputChannel = "jobChannel")
    JobLaunchRequest adapt(final String payload)  {
        final JobParameters jobParameters = new JobParametersBuilder().addString(
                "payload", payload).addDate("date", new Date()).toJobParameters();
        return new JobLaunchRequest(goJob(inactiveJobStep()), jobParameters);
    }

    @Bean
    @ServiceActivator(inputChannel = "jobChannel")
    JobLaunchingGateway sampleJobLaunchingGateway() {
        JobLaunchingGateway jobLaunchingGateway = new JobLaunchingGateway(jobLauncher);
        return jobLaunchingGateway;
    }

    @Bean
    public PollableChannel tailChannel(){
        return new QueueChannel();
    }

    @Bean
    public IntegrationFlow integrationFlow() {
        return IntegrationFlows.from(
                  Files.tailAdapter(new File("/var/log/syslog"))
                  //Files.tailAdapter(new File("/Users/juyeong/Downloads/hello/hello.txt")) //local test
                        .delay(1000)
                        .end(false) //If true, tail from the end of the file, otherwise include all lines from the beginning.
                        .reopen(true)
                        .fileDelay(1000)
                        .id("tailer")
                        .autoStartup(true)
                )
                .log()
                .filter(String.class, s -> s.contains("장비이름")) //장비이름관련된 로그만 가져올 수 있도록 필터링 처리
                .channel(tailChannel())
                .get();
    }

    @Bean
    public Job goJob(Step inactiveJobStep) {
        return jobBuilderFactory.get("goJob")
                .start(inactiveJobStep)
                .build();
    }

   @Bean
   public Step inactiveJobStep(){
        return stepBuilderFactory.get("inactiveJobStep")
                .<SyslogDataFormat, SyslogDataFormat> chunk(10) //쓰기 시에 청크 단위로 writer메서드를 실행시킬 단위지정(커밋단위가 10)
                .reader(itemReader(null))
                .writer(inactiveUserWriter())
                .build();
   }

    @Bean
    @StepScope
    public ItemWriter<SyslogDataFormat> inactiveUserWriter() {

        return (List<? extends SyslogDataFormat> list) -> list.forEach( item -> {
           log.info("received Log : {}", item);
           syslogInsertMapper.insertSyslog(item);

           String seqId = syslogInsertMapper.selectSeqId();

           if(item.getFullmsg().contains("새로운 노드 등록됨")){
               try {
                   syslogService.kafkaProducer(seqId);
                    log.info(" new node seqId : {}", seqId);

               } catch (Exception e) {
                   log.debug("kafkaProducer 중 오류 : {}", e.getMessage());
               }
           }
           log.info("detected new Node log : {}", item.getFullmsg());
        });

    }

    @Bean
    @StepScope
    public FlatFileItemReader<SyslogDataFormat> itemReader(@Value("#{jobParameters[payload]}") String resource){
        FlatFileItemReader<SyslogDataFormat> reader = new FlatFileItemReader<>();
        /*FileSystemResource fileSystemResource = new FileSystemResource(resource);
        reader.setResource(fileSystemResource);*/
        Resource resource1 = new ByteArrayResource(resource.getBytes(StandardCharsets.UTF_8));
        reader.setResource(resource1);

        log.info("converted log(utf-8) : {}", resource1);
        //String fileName = fileSystemResource.getFilename();

        DefaultLineMapper lineMapper = new DefaultLineMapper<>();
        DelimitedLineTokenizer delimitedLineTokenizer = new DelimitedLineTokenizer();
        delimitedLineTokenizer.setDelimiter("|");
        delimitedLineTokenizer.setStrict(true);
        lineMapper.setLineTokenizer(delimitedLineTokenizer);
        lineMapper.setFieldSetMapper(new SyslogFileDataFieldSetMapper(null));

        reader.setLineMapper(lineMapper);

        return reader;
    }
}

``` 
