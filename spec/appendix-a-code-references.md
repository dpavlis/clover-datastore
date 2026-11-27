# Appendix A: code references


| Topic | Path |
|---|---|
| DM configuration and datasource | `S/datamanager/DataManagerConfiguration.java`, `S/datamanager/DataManagerLocation.java`, `S/datasource/DataManagerDataSource.java` |
| DM persistence interface and implementation | `S/datamanager/DataManagerPersistence.java`, `S/datamanager/LocalDataManagerPersistence.java`, `RemoteDataManagerPersistence.java` |
| Dialect DDL | `S/datamanager/schema/SchemaHandler.java` (+ `Postgre`/`Mssql`/`Oracle`), `LayoutDiff.java`, `SystemColumn.java` |
| DM DB patches | `S/dbschema/DataManagerDbSchemaUpdate.java`, `cloveretl.server/web/WEB-INF/dbpatches/data_manager_db/` |
| DM entity | `S/persistent/DataSet.java` (table `data_set`, SINGLE_TABLE) |
| DM maintenance task | `S/persistent/Task.java` (`data_manager_maintenance`), `S/tasks/TaskProcessorDataManager.java` |
| Authority proxy dataset calls | `E/graph/runtime/IAuthorityProxy.java`, `S/graph/runtime/ServerAuthorityProxy.java`, `S/worker/ServerAuthorityProxyApiImpl.java` |
| DM components and lookup | `cloveretl.component.commercial/plugin.xml` (DataSet readers and writers), `cloveretl.lookup.commercial/src/com/opensys/cloveretl/lookup/ReferenceLookupTable.java` |
| Run tree | `S/persistent/RunRecord.java`, `cloveretl.commons/src/com/cloveretl/commons/execution/AdditionalJobProperties.java` (`ROOT_RUN_ID`) |
| Job release and events | `S/job/graph/TransformationLifeCycleStrategy.java`, `S/events/`, `S/graph/LocalExecutionManager.java` (`solveUnresolvedGraphs`) |
| Temp space | `S/tempmgr/TempFileManager.java` |
| Wrangler sources and cache | `E/wrangler/job/model/source/WConnectorSource.java`, `E/wrangler/SourceCacheProvider.java`, `S/wrangler/WorkspaceCacheProvider.java`, `S/timing/GarbageManJob.java` |
| Wrangler lookup | `E/wrangler/job/model/step/WLookup.java`, `cloveretl.component/src/org/jetel/component/HashJoin.java` (`slaveDuplicates` default `false`: first record wins) |
| Wrangler conversion context | `E/wrangler/job/model/JobModelConverter.java` (`ConversionContext`, `convertSources`), `E/wrangler/job/model/source/WSource.java` (cache writer and reader), `E/wrangler/job/model/step/WSourceStep.java` |
| Wrangler execution translation | `S/wrangler/WranglerManager.java` (`translateJobForExecution`, `loadWModelAttributes`, `translateJobToSubgraph`) |
| Connector source chain | `E/wrangler/job/model/source/WConnectorSource.java`, `SubgraphSourceTypeConverter.txt` (resource of `E/wrangler/ctl/WranglerCTLGenerator`) |
| Connector permissions | `S/library/DataConnector.java` (execute and read groups) |
| Salesforce deletes | `cloveretl.component.commercial/src/com/opensys/cloveretl/component/AbstractSalesforceReader.java` (`includeDeleted`), `SalesforceReader.java` / `SalesforceBulkReader.java` (`queryAll`) |
| Wrangler orchestration and preview | `S/wrangler/WranglerManager.java`, `S/facade/api/resp/WranglerDataset.java` |
| Connectors | `S/library/LibraryManager.java`, `S/library/DataConnectors.java`, `cloveretl.engine/src/org/jetel/library/LibraryUtils.java` |
| Data Catalog | `S/datacatalog/DataCatalog.java` |
| Engine JDBC dialects | `E/database/sql/JdbcSpecific.java`, `cloveretl.jdbc/src/org/jetel/connection/jdbc/specific/impl/` |
| Calcite and JSqlParser | `cloveretl.server/build.gradle`, `version.properties` (`calciteVersion=1.41.0`, `jsqlparserVersion=5.3`), `S/mcp/tool/utils/PerformanceLogsManager.java` |
| Licensing and permissions | `S/security/LicenseModules.java`, `S/license/LicenseBean.java` (`LicenseMaxUserCounts`), `S/facade/api/misc/Operation.java` |
| Root run id on each start path | `S/graph/runtime/ServerAuthorityProxy.java` (`prepareExecutionCommand`, `prepareWranglerExecutionCommand`), `S/job/cluster/MasterStartingStrategy.java`, `S/job/wrangler/WranglerJobStartingStrategy.java`, `S/graph/RunRecordHelper.java`, `S/graph/ExecutionHelper.java` (`ROOT_RUN_ID` parameter) |
| Final run states | `E/graph/Result.java` (`isStop()`) |
| Data service trees | `S/job/queue/JobQueue.java` (`isChildOfDataService`) |
| Per-job configuration | `S/graph/runtime/GraphConfigManager.java` (`getPropertyValue`), `cloveretl.engine/src/org/jetel/jobconfig/PropertyKey.java`, `S/persistent/JobConfig.java` (table `graph_config`) |
| Execution tree view | `S/web/gui/ExecutionTreeController.java` |
| Worker datasources and classpath | `cloveretl.server/web/WEB-INF/config.properties` (`worker.classpath`, `worker.jndi.datasource[n].*`) |
| Local runs | `cloveretl.gui/src/com/cloveretl/gui/utils/GuiAuthorityProxy.java`, `E/graph/runtime/PrimitiveAuthorityProxy.java` |
| Sandbox and workspace permissions | `S/persistent/SandboxPermission.java` (`read`, `write`, `execute`), `S/persistent/WorkspacePermissionType.java` |
| Shared storage foundation sources | `S/datamanager/schema/SchemaHandler.java` and subclasses, `S/datamanager/{Postgre,Oracle,Mssql}ExceptionTranslator.java`, `S/dbschema/DbSchemaUpdate.java` |
| Date writing in the DM | `S/datamanager/schema/SchemaHandler.java` (`toTimestamp`, `timestampType`) |

---

← [13. Open questions](13-questions.md) · [Index](README.md)
