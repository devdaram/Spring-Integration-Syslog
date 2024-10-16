<h3>💻 사용 언어 환경</h3>
  
    1. Java 21
    2. Springboot 3.3.1

<h1>✨ 맡은 업무 설명</h1>
<h3> Syslog 모듈 개발 </h3>
  
     - Spring Integration + Spring batch를 활용하여 장비 노드의 Log 메세지를 가져올 수 있는 Syslog 모듈을 개발

```java
//코드일부 수정발췌
//지속적으로 들어오는 메세지를 수집하여 파일로 생성하는 기능
public class SyslogConfig extends DefaultBatchConfigurer {

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
                  Files.tailAdapter(new File("{파일위치}"))
                        .delay(1000)
                        .end(false)
                        .reopen(true)
                        .fileDelay(1000)
                        .id("tailer")
                        .autoStartup(true)
                )
                .log()
                .filter({필터조건}) 
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

           if(item.getFullmsg().contains("{특정 토픽}")){
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
}

``` 
