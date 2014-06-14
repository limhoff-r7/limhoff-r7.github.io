---
categories: git
layout: post
title: "Preserving git history when backporting features"
---

I needed to backport the `MetasploitDataModels::Search` feature from the new module cache so the team can use it on the
search in the next release, which won't contain the new module cache.  Everything I could find on Google could get the
file, from a branch with `git checkout <branch> <file>`, but that meant the history gets lost, from `<branch>`.  That's
when I realized, that I don't care about preserving the history from `master`, since there won't be any changes relative
to `master`, except those from the branch whose history I'm preserving, so I could use `git checkout <branch> <file>`
trick, but use `master` for `<branch>` and layer it on top of the a branch off the feature branch to remove all code
that I didn't want to backport.

Here's how I did it.


Start with a branch off of the branch with the features you want

```sh
(master) > git checkout feature/exploit
(feature/exploit) > git checkout -b feature/MSP-10016/metasploit-data-models-search
```

Check the differences between the backport branch and the target branch (`master`) using `git diff`:

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --name-status master..
```

This is everything that’s changed, but I don't want to restore all of `master`, I want preserve the feature to be
backported, `lib/metasploit_data_models/search` and its accompanying specs.  First I’ll attempt just restoring all of
`master`.  This will restore files to their unmodified state from `master`, without deleting files that
`feature/MSP-10016/metasploit-data-models-search` had added in its history.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git checkout master -- .
(feature/MSP-10016/metasploit-data-models-search) > git status
```

So, I have a bunch of `'modified'` and `'new file'` statuses.  The `‘new file’` status is from files that exist in
`master`, but were deleted in `feature/MSP-10016/metasploit-data-models-search` and are now being restored.  The
`git checkout master — .` added all the files to the index, so I now need to use `git diff --cached` as `--cached` will
diff against the index..

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master
```

There are added models under `app/models/mdm` that need to be removed because they're from the new module cache, but I
want to keep `app/models/metasploit_data_models`, so I'll delete `app/models/mdm` added files:

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master app/models/mdm`
rm 'app/models/mdm/architecture.rb'
rm 'app/models/mdm/author.rb'
rm 'app/models/mdm/authority.rb'
rm 'app/models/mdm/email_address.rb'
rm 'app/models/mdm/module/ancestor.rb'
rm 'app/models/mdm/module/architecture.rb'
rm 'app/models/mdm/module/class.rb'
rm 'app/models/mdm/module/instance.rb'
rm 'app/models/mdm/module/path.rb'
rm 'app/models/mdm/module/rank.rb'
rm 'app/models/mdm/module/reference.rb'
rm 'app/models/mdm/module/relationship.rb'
rm 'app/models/mdm/module/target/architecture.rb'
rm 'app/models/mdm/module/target/platform.rb'
rm 'app/models/mdm/platform.rb'
rm 'app/models/mdm/reference.rb'
rm 'app/models/mdm/report.rb'
rm 'app/models/mdm/report_template.rb'
rm 'app/models/mdm/vuln_reference.rb'
```

Double check that `app/models/mdm` matches `master` now

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master app/models
A       app/models/metasploit_data_models/search/visitor/attribute.rb
A       app/models/metasploit_data_models/search/visitor/includes.rb
A       app/models/metasploit_data_models/search/visitor/joins.rb
A       app/models/metasploit_data_models/search/visitor/method.rb
A       app/models/metasploit_data_models/search/visitor/relation.rb
A       app/models/metasploit_data_models/search/visitor/where.rb
```

Let's check what else is left that's different than `master`

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master
A     app/models/metasploit_data_models/search/visitor/attribute.rb
A     app/models/metasploit_data_models/search/visitor/includes.rb
A     app/models/metasploit_data_models/search/visitor/joins.rb
A     app/models/metasploit_data_models/search/visitor/method.rb
A     app/models/metasploit_data_models/search/visitor/relation.rb
A     app/models/metasploit_data_models/search/visitor/where.rb
A     config/locales/en.yml
A     db/migrate/20130524210148_create_module_paths.rb
A     db/migrate/20130528183122_add_parent_path_to_module_details.rb
A     db/migrate/20130530222124_change_column_null_in_module_actions.rb
A     db/migrate/20130531005804_unique_name_scopedto_detail_id_in_module_actions.rb
A     db/migrate/20130531140620_change_column_null_in_module_archs.rb
A     db/migrate/20130531161050_unique_name_scoped_to_detail_id_in_module_archs.rb
A     db/migrate/20130531175202_change_required_columns_to_null_false_in_module_authors.rb
A     db/migrate/20130531185216_unique_name_scoped_to_detail_id_in_module_authors.rb
A     db/migrate/20130602182516_create_module_ancestors.rb
A     db/migrate/20130602202346_create_module_ranks.rb
A     db/migrate/20130602202347_create_module_classes.rb
A     db/migrate/20130602214406_create_module_relationships.rb
A     db/migrate/20130602230424_create_module_instances.rb
A     db/migrate/20130612134845_associate_module_actions_to_module_instances.rb
A     db/migrate/20130612155900_create_architectures.rb
A     db/migrate/20130612213434_change_hosts_arch_to_architecture_id.rb
A     db/migrate/20130613131314_replace_module_archs_with_module_architectures.rb
A     db/migrate/20130613152322_drop_module_mixins.rb
A     db/migrate/20130613200029_create_platforms.rb
A     db/migrate/20130614130338_change_module_platforms_to_join_model.rb
A     db/migrate/20130614181004_create_authorities.rb
A     db/migrate/20130617151106_create_references.rb
A     db/migrate/20130618150940_create_module_references.rb
A     db/migrate/20130618163932_create_vuln_references.rb
A     db/migrate/20130618185051_drop_module_refs.rb
A     db/migrate/20130618190204_translate_refs_to_references.rb
A     db/migrate/20130618204649_drop_refs.rb
A     db/migrate/20130618205631_drop_vulns_refs.rb
A     db/migrate/20130619020112_create_authors.rb
A     db/migrate/20130619195002_create_email_addresses.rb
A     db/migrate/20130619204107_repurpose_module_authors.rb
A     db/migrate/20130619212332_reference_module_instances_in_module_targets.rb
A     db/migrate/20130619215217_drop_module_details.rb
A     db/migrate/20130620182424_rename_hosts_tags_to_host_tags.rb
A     db/migrate/20130620194037_unique_host_tags.rb
A     db/migrate/20130621173328_change_required_columns_to_null_false_in_api_keys.rb
A     db/migrate/20130621181259_unique_token_in_api_keys.rb
A     db/migrate/20130626183210_drop_mod_refs.rb
A     db/migrate/20130627175028_unique_task_creds.rb
A     db/migrate/20130629144839_unique_task_hosts.rb
A     db/migrate/20130629164610_unique_task_services.rb
A     db/migrate/20130629173534_unique_task_sessions.rb
A     db/migrate/20131009212856_add_full_to_email_address.rb
A     db/migrate/20131015191312_nested_set_platforms.rb
A     db/migrate/20131028181307_create_module_target_architectures.rb
A     db/migrate/20131028181807_create_module_target_platforms.rb
A     db/migrate/20131030182452_remove_index_from_module_targets.rb
A     db/migrate/20140106022804_add_module_class_id_to_exploit_attempts.rb
A     db/migrate/20140106180149_replace_platform_with_architecture_id_and_platform_id_in_sessions.rb
A     db/migrate/20140107170605_add_exploit_class_id_and_payload_class_id_to_sessions.rb
A     db/migrate/20140108154835_add_module_class_id_to_vuln_attempts.rb
A     db/migrate/20140110145215_changed_required_columns_to_null_false_in_exploit_attempts.rb
A     db/migrate/20140110145947_changed_required_columns_to_null_false_in_vuln_attempts.rb
A     db/seeds.rb
A     docs/images/.gitkeep
A     docs/mdm_module_sql_translation.md
A     lib/metasploit_data_models/attempt.rb
A     lib/metasploit_data_models/batch.rb
A     lib/metasploit_data_models/batch/descendant.rb
A     lib/metasploit_data_models/batch/root.rb
A     lib/metasploit_data_models/entity_relationship_diagram.rb
A     lib/metasploit_data_models/entity_relationship_diagram/module.rb
A     lib/metasploit_data_models/null_progress_bar.rb
A     lib/metasploit_data_models/search.rb
A     lib/metasploit_data_models/search/visitor.rb
A     lib/metasploit_data_models/unique_task_joins.rb
A     lib/tasks/erd.rake
A     spec/app/models/mdm/api_key_spec.rb
A     spec/app/models/mdm/architecture_spec.rb
A     spec/app/models/mdm/author_spec.rb
A     spec/app/models/mdm/authority_spec.rb
A     spec/app/models/mdm/email_address_spec.rb
A     spec/app/models/mdm/module/ancestor_spec.rb
A     spec/app/models/mdm/module/architecture_spec.rb
A     spec/app/models/mdm/module/class_spec.rb
A     spec/app/models/mdm/module/instance_spec.rb
A     spec/app/models/mdm/module/path_spec.rb
A     spec/app/models/mdm/module/rank_spec.rb
A     spec/app/models/mdm/module/reference_spec.rb
A     spec/app/models/mdm/module/relationship_spec.rb
A     spec/app/models/mdm/module/target/architecture_spec.rb
A     spec/app/models/mdm/module/target/platform_spec.rb
A     spec/app/models/mdm/platform_spec.rb
A     spec/app/models/mdm/reference_spec.rb
A     spec/app/models/mdm/report_spec.rb
A     spec/app/models/mdm/report_template_spec.rb
A     spec/app/models/mdm/task_cred_spec.rb
A     spec/app/models/mdm/task_session_spec.rb
A     spec/app/models/mdm/vuln_detail_spec.rb
A     spec/app/models/mdm/vuln_reference_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/joins_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/method_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/relation_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/where_spec.rb
A     spec/db/seeds_spec.rb
A     spec/dummy/config/initializers/active_record_migrations.rb
A     spec/factories/mdm/api_keys.rb
A     spec/factories/mdm/architectures.rb
A     spec/factories/mdm/attempts.rb
A     spec/factories/mdm/authorities.rb
A     spec/factories/mdm/authors.rb
A     spec/factories/mdm/email_addresses.rb
A     spec/factories/mdm/module/ancestors.rb
A     spec/factories/mdm/module/architectures.rb
A     spec/factories/mdm/module/classes.rb
A     spec/factories/mdm/module/instances.rb
A     spec/factories/mdm/module/paths.rb
A     spec/factories/mdm/module/ranks.rb
A     spec/factories/mdm/module/references.rb
A     spec/factories/mdm/module/relationships.rb
A     spec/factories/mdm/module/target/architectures.rb
A     spec/factories/mdm/module/target/platforms.rb
A     spec/factories/mdm/platforms.rb
A     spec/factories/mdm/references.rb
A     spec/factories/mdm/report_templates.rb
A     spec/factories/mdm/reports.rb
A     spec/factories/mdm/vuln_references.rb
A     spec/lib/metasploit_data_models/base64_serializer_spec.rb
A     spec/lib/metasploit_data_models/batch/descendant_spec.rb
A     spec/lib/metasploit_data_models/batch/root_spec.rb
A     spec/lib/metasploit_data_models/batch_spec.rb
A     spec/support/shared/contexts/active_record/attribute_type.rb
A     spec/support/shared/contexts/dummy_mdm_module_path.rb
A     spec/support/shared/contexts/metasploit_data_models/batch/batch.rb
A     spec/support/shared/examples/mdm/architecture/seed.rb
A     spec/support/shared/examples/mdm/attempt.rb
A     spec/support/shared/examples/mdm/module/path/changed_module_ancestor_from_real_path/with_handled.rb
A     spec/support/shared/examples/mdm/module/path/changed_module_ancestor_from_real_path/without_handled.rb
A     spec/support/shared/examples/metasploit_data_models/batch/descendant.rb
A     spec/support/shared/examples/metasploit_data_models/batch/root.rb
A     spec/support/shared/examples/metasploit_data_models/db/seeds.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_children.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_metasploit_model_search_operation_base.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_opeator_deprecated_platform.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_operator_deprecated_authority.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_equality.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_metasploit_model_search_group_base.rb
```

I'll skip `config/locales.en.yml` for now as I may need to do line edits there, so the next directory to clean up is
`db`.  Any new migrations or seeds can be ignored so I can just do the selective delete again

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master db`
rm 'db/migrate/20130524210148_create_module_paths.rb'
rm 'db/migrate/20130528183122_add_parent_path_to_module_details.rb'
rm 'db/migrate/20130530222124_change_column_null_in_module_actions.rb'
rm 'db/migrate/20130531005804_unique_name_scopedto_detail_id_in_module_actions.rb'
rm 'db/migrate/20130531140620_change_column_null_in_module_archs.rb'
rm 'db/migrate/20130531161050_unique_name_scoped_to_detail_id_in_module_archs.rb'
rm 'db/migrate/20130531175202_change_required_columns_to_null_false_in_module_authors.rb'
rm 'db/migrate/20130531185216_unique_name_scoped_to_detail_id_in_module_authors.rb'
rm 'db/migrate/20130602182516_create_module_ancestors.rb'
rm 'db/migrate/20130602202346_create_module_ranks.rb'
rm 'db/migrate/20130602202347_create_module_classes.rb'
rm 'db/migrate/20130602214406_create_module_relationships.rb'
rm 'db/migrate/20130602230424_create_module_instances.rb'
rm 'db/migrate/20130612134845_associate_module_actions_to_module_instances.rb'
rm 'db/migrate/20130612155900_create_architectures.rb'
rm 'db/migrate/20130612213434_change_hosts_arch_to_architecture_id.rb'
rm 'db/migrate/20130613131314_replace_module_archs_with_module_architectures.rb'
rm 'db/migrate/20130613152322_drop_module_mixins.rb'
rm 'db/migrate/20130613200029_create_platforms.rb'
rm 'db/migrate/20130614130338_change_module_platforms_to_join_model.rb'
rm 'db/migrate/20130614181004_create_authorities.rb'
rm 'db/migrate/20130617151106_create_references.rb'
rm 'db/migrate/20130618150940_create_module_references.rb'
rm 'db/migrate/20130618163932_create_vuln_references.rb'
rm 'db/migrate/20130618185051_drop_module_refs.rb'
rm 'db/migrate/20130618190204_translate_refs_to_references.rb'
rm 'db/migrate/20130618204649_drop_refs.rb'
rm 'db/migrate/20130618205631_drop_vulns_refs.rb'
rm 'db/migrate/20130619020112_create_authors.rb'
rm 'db/migrate/20130619195002_create_email_addresses.rb'
rm 'db/migrate/20130619204107_repurpose_module_authors.rb'
rm 'db/migrate/20130619212332_reference_module_instances_in_module_targets.rb'
rm 'db/migrate/20130619215217_drop_module_details.rb'
rm 'db/migrate/20130620182424_rename_hosts_tags_to_host_tags.rb'
rm 'db/migrate/20130620194037_unique_host_tags.rb'
rm 'db/migrate/20130621173328_change_required_columns_to_null_false_in_api_keys.rb'
rm 'db/migrate/20130621181259_unique_token_in_api_keys.rb'
rm 'db/migrate/20130626183210_drop_mod_refs.rb'
rm 'db/migrate/20130627175028_unique_task_creds.rb'
rm 'db/migrate/20130629144839_unique_task_hosts.rb'
rm 'db/migrate/20130629164610_unique_task_services.rb'
rm 'db/migrate/20130629173534_unique_task_sessions.rb'
rm 'db/migrate/20131009212856_add_full_to_email_address.rb'
rm 'db/migrate/20131015191312_nested_set_platforms.rb'
rm 'db/migrate/20131028181307_create_module_target_architectures.rb'
rm 'db/migrate/20131028181807_create_module_target_platforms.rb'
rm 'db/migrate/20131030182452_remove_index_from_module_targets.rb'
rm 'db/migrate/20140106022804_add_module_class_id_to_exploit_attempts.rb'
rm 'db/migrate/20140106180149_replace_platform_with_architecture_id_and_platform_id_in_sessions.rb'
rm 'db/migrate/20140107170605_add_exploit_class_id_and_payload_class_id_to_sessions.rb'
rm 'db/migrate/20140108154835_add_module_class_id_to_vuln_attempts.rb'
rm 'db/migrate/20140110145215_changed_required_columns_to_null_false_in_exploit_attempts.rb'
rm 'db/migrate/20140110145947_changed_required_columns_to_null_false_in_vuln_attempts.rb'
rm 'db/seeds.rb'
```

I also know that the additions to the `docs` directory are unneeded since they document upgrade steps for the module
cache.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master docs`
rm 'docs/images/.gitkeep'
rm 'docs/mdm_module_sql_translation.md'
```

Now I need to look at what's left in `lib/metasploit_data_models` to determine what needs to be deleted.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master lib/metasploit_data_models
A     lib/metasploit_data_models/attempt.rb
A     lib/metasploit_data_models/batch.rb
A     lib/metasploit_data_models/batch/descendant.rb
A     lib/metasploit_data_models/batch/root.rb
A     lib/metasploit_data_models/entity_relationship_diagram.rb
A     lib/metasploit_data_models/entity_relationship_diagram/module.rb
A     lib/metasploit_data_models/null_progress_bar.rb
A     lib/metasploit_data_models/search.rb
A     lib/metasploit_data_models/search/visitor.rb
A     lib/metasploit_data_models/unique_task_joins.rb
```

`lib/metasploit_data_models/attempt.rb` is common code between `Mdm::ExploitAttempt` and `Mdm::VulnAttempt`, but I'm
not taking that refactor, so it can be deleted.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm lib/metasploit_data_models/attempt.rb
rm 'lib/metasploit_data_models/attempt.rb'
```

`lib/metasploit_data_models/batch*` is part of the batching system that disables costly per record uniqueness
validations for the module cache, so it's out.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm -r lib/metasploit_data_models/batch*
rm 'lib/metasploit_data_models/batch.rb'
rm 'lib/metasploit_data_models/batch/descendant.rb'
rm 'lib/metasploit_data_models/batch/root.rb'
```

The ERD system has been moved to `metasploit-erd`, but I don't want that part of this backport, so I'll remove
`lib/metasploit_data_models/entity_relationship_diagram*`.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm -r lib/metasploit_data_models/entity_relationship_diagram*
rm 'lib/metasploit_data_models/entity_relationship_diagram.rb'
rm 'lib/metasploit_data_models/entity_relationship_diagram/module.rb'
```

While I'm at it, I'll also remove the rake task

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm lib/tasks/erd.rake
rm 'lib/tasks/erd.rake'
```

`NullProgressBar` is part progress bar system for `Mdm::Module::Path`, which I already removed, so
`lib/metasploit_data_models/null_progress_bar.rb` can be removed too.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm lib/metasploit_data_models/null_progress_bar.rb
rm 'lib/metasploit_data_models/null_progress_bar.rb'
```

I'll obvious want to keep `lib/metasploit_data_models/search*`, so that just leaves
`lib/metasploit_data_models/unique_task_joins.rb`.  I don't need it because it's part of migration refactors I
already threw away.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm lib/metasploit_data_models/unique_task_joins.rb
rm 'lib/metasploit_data_models/unique_task_joins.rb'
```

Let's just check `lib/metasploit_data_models` for any stragglers:

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master lib/metasploit_data_models
A     lib/metasploit_data_models/search.rb
A     lib/metasploit_data_models/search/visitor.rb
```

So, now `app` and `lib` are good, but that means spec needs to be cleaned.  I could just iteratively run the specs
until they pass, but `rspec` won't run with undefined classes in the `describe` block, so let's mirror the deletes I
did in `app` and `lib` in `spec`:

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/app/models/mdm`
rm 'spec/app/models/mdm/api_key_spec.rb'
rm 'spec/app/models/mdm/architecture_spec.rb'
rm 'spec/app/models/mdm/author_spec.rb'
rm 'spec/app/models/mdm/authority_spec.rb'
rm 'spec/app/models/mdm/email_address_spec.rb'
rm 'spec/app/models/mdm/module/ancestor_spec.rb'
rm 'spec/app/models/mdm/module/architecture_spec.rb'
rm 'spec/app/models/mdm/module/class_spec.rb'
rm 'spec/app/models/mdm/module/instance_spec.rb'
rm 'spec/app/models/mdm/module/path_spec.rb'
rm 'spec/app/models/mdm/module/rank_spec.rb'
rm 'spec/app/models/mdm/module/reference_spec.rb'
rm 'spec/app/models/mdm/module/relationship_spec.rb'
rm 'spec/app/models/mdm/module/target/architecture_spec.rb'
rm 'spec/app/models/mdm/module/target/platform_spec.rb'
rm 'spec/app/models/mdm/platform_spec.rb'
rm 'spec/app/models/mdm/reference_spec.rb'
rm 'spec/app/models/mdm/report_spec.rb'
rm 'spec/app/models/mdm/report_template_spec.rb'
rm 'spec/app/models/mdm/task_cred_spec.rb'
rm 'spec/app/models/mdm/task_session_spec.rb'
rm 'spec/app/models/mdm/vuln_detail_spec.rb'
rm 'spec/app/models/mdm/vuln_reference_spec.rb'
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/app/models/mdm`
rm 'spec/db/seeds_spec.rb'
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/factories/mdm`
rm 'spec/factories/mdm/api_keys.rb'
rm 'spec/factories/mdm/architectures.rb'
rm 'spec/factories/mdm/attempts.rb'
rm 'spec/factories/mdm/authorities.rb'
rm 'spec/factories/mdm/authors.rb'
rm 'spec/factories/mdm/email_addresses.rb'
rm 'spec/factories/mdm/module/ancestors.rb'
rm 'spec/factories/mdm/module/architectures.rb'
rm 'spec/factories/mdm/module/classes.rb'
rm 'spec/factories/mdm/module/instances.rb'
rm 'spec/factories/mdm/module/paths.rb'
rm 'spec/factories/mdm/module/ranks.rb'
rm 'spec/factories/mdm/module/references.rb'
rm 'spec/factories/mdm/module/relationships.rb'
rm 'spec/factories/mdm/module/target/architectures.rb'
rm 'spec/factories/mdm/module/target/platforms.rb'
rm 'spec/factories/mdm/platforms.rb'
rm 'spec/factories/mdm/references.rb'
rm 'spec/factories/mdm/report_templates.rb'
rm 'spec/factories/mdm/reports.rb'
rm 'spec/factories/mdm/vuln_references.rb'
```

I can safely delete `spec/lib/metasploit_data_models` because the there are no specs for
`lib/metasploit_data_models` code I want to preserve because they are just namespace `Module`s.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/lib/metasploit_data_models`
rm 'spec/lib/metasploit_data_models/base64_serializer_spec.rb'
rm 'spec/lib/metasploit_data_models/batch/descendant_spec.rb'
rm 'spec/lib/metasploit_data_models/batch/root_spec.rb'
rm 'spec/lib/metasploit_data_models/batch_spec.rb'
```

Thankfully, most of the shared examples have names that match the classes for which they are used, so we can delete
shared examples for classes I didn't backport

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/support/shared/examples/mdm`
rm 'spec/support/shared/examples/mdm/architecture/seed.rb'
rm 'spec/support/shared/examples/mdm/attempt.rb'
rm 'spec/support/shared/examples/mdm/module/path/changed_module_ancestor_from_real_path/with_handled.rb'
rm 'spec/support/shared/examples/mdm/module/path/changed_module_ancestor_from_real_path/without_handled.rb'
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/support/shared/examples/metasploit_data_models/batch`
rm 'spec/support/shared/examples/metasploit_data_models/batch/descendant.rb'
rm 'spec/support/shared/examples/metasploit_data_models/batch/root.rb'
(feature/MSP-10016/metasploit-data-models-search) > git rm spec/support/shared/examples/metasploit_data_models/db/seeds.rb
rm 'spec/support/shared/examples/metasploit_data_models/db/seeds.rb'
```

Double check that only `MetasploitDataModels::Search`` shared examples are left

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master spec/support/shared/examples
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_children.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_metasploit_model_search_operation_base.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_opeator_deprecated_platform.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_operator_deprecated_authority.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_equality.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_metasploit_model_search_group_base.rb
```

Moving on to the shared contexts, `dummy_mdm_module_path.rb` is obvious out because I removed `Mdm::Module::Path`.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm spec/support/shared/contexts/dummy_mdm_module_path.rb
rm 'spec/support/shared/contexts/dummy_mdm_module_path.rb'
```

Let's see what's left in shared contexts:

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master spec/support/shared/contexts
A     spec/support/shared/contexts/active_record/attribute_type.rb
A     spec/support/shared/contexts/metasploit_data_models/batch/batch.rb
```

Batch is out

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm spec/support/shared/contexts/metasploit_data_models/batch/batch.rb
rm 'spec/support/shared/contexts/metasploit_data_models/batch/batch.rb'
```

I'm not sure what `'ActiveRecord attribute_type'` is, so let's search for it.

```sh
(feature/MSP-10016/metasploit-data-models-search) > ack "ActiveRecord attribute_type"
spec/support/shared/contexts/active_record/attribute_type.rb
1:shared_context 'ActiveRecord attribute_type' do
```

So, no uses, delete it.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm spec/support/shared/contexts/active_record/attribute_type.rb
rm 'spec/support/shared/contexts/active_record/attribute_type.rb'
```

Let's see what I have left with

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master
A     app/models/metasploit_data_models/search/visitor/attribute.rb
A     app/models/metasploit_data_models/search/visitor/includes.rb
A     app/models/metasploit_data_models/search/visitor/joins.rb
A     app/models/metasploit_data_models/search/visitor/method.rb
A     app/models/metasploit_data_models/search/visitor/relation.rb
A     app/models/metasploit_data_models/search/visitor/where.rb
A     config/locales/en.yml
A     lib/metasploit_data_models/search.rb
A     lib/metasploit_data_models/search/visitor.rb
A     spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/joins_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/method_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/relation_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/where_spec.rb
A     spec/dummy/config/initializers/active_record_migrations.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_children.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_metasploit_model_search_operation_base.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_opeator_deprecated_platform.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_operator_deprecated_authority.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_equality.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_metasploit_model_search_group_base.rb
```

`config/locals/en.yml` and `spec/dummy/config/initializers/active_record_migrations.rb` might not be needed, but I'm not
sure, so let's test.

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake db:drop db:create db:migrate
(feature/MSP-10016/metasploit-data-models-search) > rake spec
...
/Users/luke.imhoff/git/rapid7/metasploit_data_models/app/models/metasploit_data_models/search/visitor/attribute.rb:3:in `<class:Attribute>': uninitialized constant MetasploitDataModels::Search::Visitor::Attribute::Metasploit (NameError)
...
```

Whoops, I lost `metasploit-model` because I replaced the the `gemspec` and `Gemfile` with the one from `master`, which
doesn't know about `metasploit-model`.  So, I'll add `metasploit-model` back to the `gemspec`.

```
diff --git a/metasploit_data_models.gemspec b/metasploit_data_models.gemspec
index 71fa7d2..6354403 100644
--- a/metasploit_data_models.gemspec
+++ b/metasploit_data_models.gemspec
@@ -38,6 +38,7 @@ Gem::Specification.new do |s|
   # @see MSP-2971
   s.add_runtime_dependency 'activerecord', '>= 3.2.13', '< 4.0.0'
   s.add_runtime_dependency 'activesupport'
+  s.add_runtime_dependency 'metasploit-model', '>= 0.24.1.pre.semantic.pre.versioning.pre.2.pre.0', '< 0.25'
  
   if RUBY_PLATFORM =~ /java/
     # markdown formatting for yard
```

```sh
(feature/MSP-10016/metasploit-data-models-search) > bundle install
...
Installing metasploit-model (0.24.1.pre.semantic.pre.versioning.pre.2.pre.0)
...
```

Try specs again

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake spec
/Users/luke.imhoff/git/rapid7/metasploit_data_models/app/models/metasploit_data_models/search/visitor/attribute.rb:3:in `<class:Attribute>': uninitialized constant MetasploitDataModels::Search::Visitor::Attribute::Metasploit (NameError)
```

`Metasploit` isn't resolving, so the code is probably missing the explicit `require` in `lib/metasploit_data_models.rb`.

diff --git a/lib/metasploit_data_models.rb b/lib/metasploit_data_models.rb
index 70d408d..936753d 100755
--- a/lib/metasploit_data_models.rb
+++ b/lib/metasploit_data_models.rb
@@ -10,6 +10,7 @@ require 'active_record'
 require 'active_support'
 require 'active_support/all'
 require 'active_support/dependencies'
+require 'metasploit/model'
 
 #
 # Project


Test again

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake spec
...
NameError: uninitialized constant Mdm::Module::Instance
...
```

Whoops, looks like the tests for `MetasploitDataModels::Search::Visitor::*` were written against the actual, new module
cache classes instead of dummy classes, which is great for behavior testing on the module caching branches, but does us
no good now where I don't yet have classes that can be searched.  Thankfully, from the spec comments I know what I
need to supply to use a class I do have:

```ruby
        Metasploit::Model::Search::Operator::Attribute.new(
            # needs to be a real column so look up on AREL table works
            :attribute => :module_type,
            # needs to be a real class so Class#arel_table works
            :klass => Mdm::Module::Instance
        )
```

Fortunately, as part of the search usage, I know the team will want to eventually make `Mdm::Host` and `Mdm::Service`
searchable, so I'll use `Mdm::Host` in place of `Mdm::Module::Instance` and `Mdm::Service` for any associations for
`Mdm::Module::Instance` to test the join and include logic.

```
diff --git a/spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb b/spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
index d031898..69dcd40 100644
--- a/spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
+++ b/spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
@@ -42,9 +42,9 @@ describe MetasploitDataModels::Search::Visitor::Attribute do
       let(:node) do
         Metasploit::Model::Search::Operator::Attribute.new(
             # needs to be a real column so look up on AREL table works
-            :attribute => :module_type,
+            :attribute => :name,
             # needs to be a real class so Class#arel_table works
-            :klass => Mdm::Module::Instance
+            :klass => Mdm::Host
         )
       end
```

Test just that file

```sh
 (feature/MSP-10016/metasploit-data-models-search) > rspec spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb

MetasploitDataModels::Search::Visitor::Attribute
  #visit
    with Metasploit::Model::Search::Operator::Attribute
      should be a kind of Arel::Attributes::Attribute
      relation
        should be Class#arel_table for Metasploit::Model::Search::Operator::Attribute#klass
      name
        should be Metasploit::Model::Search::Operator::Attribute#attribute
    with Metasploit::Model::Search::Operator::Association
      should return visit of Metasploit::Model::Search::Operator::Association#attribute_operator
      should visit Metasploit::Model::Search::Operator::Association#attribute_operator


Finished in 0.0211 seconds
5 examples, 0 failures
```

Ok, so that works.  Now, let's do the same for `spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb`

```
diff --git a/spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb b/spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
index 91ca5b2..c9a1823 100644
--- a/spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
+++ b/spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
@@ -84,316 +84,40 @@ describe MetasploitDataModels::Search::Visitor::Includes do
       let(:query) do
         Metasploit::Model::Search::Query.new(
             :formatted => formatted,
-            :klass => Mdm::Module::Instance
+            :klass => klass
         )
       end
 
-      context 'with description' do
-        let(:description) do
-          FactoryGirl.generate :metasploit_model_module_instance_description
-        end
-
-        let(:formatted) do
-          "description:\"#{description}\""
-        end
-
-        it { should be_empty }
-      end
-
-      context 'with disclosed_on' do
-        let(:disclosed_on) do
-          FactoryGirl.generate :metasploit_model_module_instance_disclosed_on
-        end
-
-        let(:formatted) do
-          "disclosed_on:\"#{disclosed_on}\""
-        end
-
-        it { should be_empty }
-      end
-
-      context 'with license' do
-        let(:license) do
-          FactoryGirl.generate :metasploit_model_module_instance_license
-        end
-
-        let(:formatted) do
-          "license:\"#{license}\""
-        end
-
-        it { should be_empty }
-      end
-
-      context 'with name' do
-        let(:name) do
-          FactoryGirl.generate :metasploit_model_module_instance_name
-        end
-
-        let(:formatted) do
-          "name:\"#{name}\""
-        end
-
-        it { should be_empty }
-      end
-
-      context 'with privileged' do
-        let(:privileged) do
-          FactoryGirl.generate :metasploit_model_module_instance_privileged
-        end
-
-        let(:formatted) do
-          "privileged:#{privileged}"
-        end
-
-        it { should be_empty }
-      end
-
-      context 'with stance' do
-        let(:stance) do
-          FactoryGirl.generate :metasploit_model_module_stance
-        end
-
-        let(:formatted) do
-          "stance:#{stance}"
-        end
-
-        it { should be_empty }
-      end
-
-      context 'with actions.name' do
-        let(:name) do
-          FactoryGirl.generate :metasploit_model_module_action_name
-        end
+      context 'Metasploit::Model::Search::Query#klass' do
+        context 'with Mdm::Host' do
+          let(:klass) {
+            Mdm::Host
+          }
 
-        let(:formatted) do
-          "actions.name:\"#{name}\""
-        end
-
-        it { should include :actions }
-      end
-
-      context 'with architectures.abbreviation' do
-        let(:abbreviation) do
-          FactoryGirl.generate :metasploit_model_architecture_abbreviation
-        end
+          context 'with name' do
+            let(:name) do
+              FactoryGirl.generate :mdm_host_name
+            end
 
-        let(:formatted) do
-          "architectures.abbreviation:#{abbreviation}"
-        end
-
-        it { should include :architectures }
-      end
-
-      context 'with architectures.bits' do
-        let(:bits) do
-          FactoryGirl.generate :metasploit_model_architecture_bits
-        end
+            let(:formatted) do
+              "name:\"#{name}\""
+            end
 
-        let(:formatted) do
-          "architectures.bits:#{bits}"
-        end
-
-        it { should include :architectures }
-      end
-
-      context 'with architectures.endianness' do
-        let(:endianness) do
-          FactoryGirl.generate :metasploit_model_architecture_endianness
-        end
-
-        let(:formatted) do
-          "architectures.endianness:#{endianness}"
-        end
-
-        it { should include :architectures }
-      end
-
-      context 'with architectures.family' do
-        let(:family) do
-          FactoryGirl.generate :metasploit_model_architecture_family
-        end
-
-        let(:formatted) do
-          "architectures.family:#{family}"
-        end
-
-        it { should include :architectures }
-      end
-
-      context 'with authorities.abbreviation' do
-        let(:abbreviation) do
-          FactoryGirl.generate :metasploit_model_authority_abbreviation
-        end
-
-        let(:formatted) do
-          "authorities.abbreviation:#{abbreviation}"
-        end
-
-        it { should include :authorities }
-      end
-
-      context 'with authors.name' do
-        let(:name) do
-          FactoryGirl.generate :metasploit_model_author_name
-        end
-
-        let(:formatted) do
-          "authors.name:\"#{name}\""
-        end
-
-        it { should include :authors }
-      end
-
-      context 'with email_addresses.domain' do
-        let(:domain) do
-          FactoryGirl.generate :metasploit_model_email_address_domain
-        end
-
-        let(:formatted) do
-          "email_addresses.domain:#{domain}"
-        end
-
-        it { should include :email_addresses }
-      end
-
-      context 'with email_addresses.local' do
-        let(:local) do
-          FactoryGirl.generate :metasploit_model_email_address_local
-        end
-
-        let(:formatted) do
-          "email_addresses.local:#{local}"
-        end
-
-        it { should include :email_addresses }
-      end
-
-      context 'with module_class.full_name' do
-        let(:full_name) do
-          "#{module_type}/#{reference_name}"
-        end
-
-        let(:formatted) do
-          "module_class.full_name:#{full_name}"
-        end
-
-        let(:module_type) do
-          FactoryGirl.generate :metasploit_model_non_payload_module_type
-        end
-
-        let(:reference_name) do
-          FactoryGirl.generate :metasploit_model_module_ancestor_non_payload_reference_name
-        end
-
-        it { should include :module_class }
-      end
-
-      context 'with module_class.module_type' do
-        let(:formatted) do
-          "module_class.module_type:#{module_type}"
-        end
-
-        let(:module_type) do
-          FactoryGirl.generate :metasploit_model_module_type
-        end
-
-        it { should include :module_class }
-      end
-
-      context 'with module_class.payload_type' do
-        let(:formatted) do
-          "module_class.payload_type:#{payload_type}"
-        end
-
-        let(:payload_type) do
-          FactoryGirl.generate :metasploit_model_module_class_payload_type
-        end
-
-        it { should include :module_class }
-      end
-
-      context 'with module_class.reference_name' do
-        let(:formatted) do
-          "module_class.reference_name:#{reference_name}"
-        end
-
-        let(:reference_name) do
-          FactoryGirl.generate :metasploit_model_module_ancestor_reference_name
-        end
-
-        it { should include :module_class }
-      end
-
-      context 'with platforms.fully_qualified_name' do
-        let(:formatted) do
-          "platforms.fully_qualified_name:\"#{fully_qualified_name}\""
-        end
-
-        let(:fully_qualified_name) do
-          Metasploit::Model::Platform.fully_qualified_name_set.to_a.sample
-        end
-
-        it { should include :platforms }
-      end
-
-      context 'with rank.name' do
-        let(:formatted) do
-          "rank.name:#{name}"
-        end
-
-        let(:name) do
-          FactoryGirl.generate :metasploit_model_module_rank_name
-        end
-
-        it { should include :rank }
-      end
-
-      context 'with rank.number' do
-        let(:formatted) do
-          "rank.number:#{number}"
-        end
-
-        let(:number) do
-          FactoryGirl.generate :metasploit_model_module_rank_number
-        end
-
-        it { should include :rank }
-      end
-
-      context 'with references.designation' do
-        let(:designation) do
-          FactoryGirl.generate :metasploit_model_reference_designation
-        end
-
-        let(:formatted) do
-          "references.designation:#{designation}"
-        end
-
-        it { should include :references }
-      end
+            it { should be_empty }
+          end
 
-      context 'with references.url' do
-        let(:formatted) do
-          "references.url:#{url}"
-        end
+          context 'with services.name' do
+            let(:name) do
+              FactoryGirl.generate :mdm_service_name
+            end
 
-        let(:url) do
-          FactoryGirl.generate :metasploit_model_reference_url
-        end
-
-        it { should include :references }
-      end
+            let(:formatted) do
+              "services.name:\"#{name}\""
+            end
 
-      context 'with targets.name' do
-        let(:formatted) do
-          "targets.name:\"#{name}\""
-        end
-
-        let(:name) do
-          FactoryGirl.generate :metasploit_model_module_target_name
+            it { should include :actions }
+          end
         end
-
-        it { should include :targets }
       end
     end
   end
```

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake spec
...
NoMethodError: undefined method `search_operator_by_name' for #<Class:0x007f86582a8c68>
...
```

Ok, so I need to define the search operators using the search DSL on `Mdm::Host` and `Mdm::Service`

```
diff --git a/app/models/mdm/host.rb b/app/models/mdm/host.rb
index a7939b1..474d0ec 100755
--- a/app/models/mdm/host.rb
+++ b/app/models/mdm/host.rb
@@ -1,6 +1,7 @@
 # A system with an {#address IP address} on the network that has been discovered in some way.
 class Mdm::Host < ActiveRecord::Base
   include Mdm::Host::OperatingSystemNormalization
+  include Metasploit::Model::Search
 
   #
   # CONSTANTS
@@ -462,6 +463,29 @@ class Mdm::Host < ActiveRecord::Base
   scope :tag_search,
         lambda { |*args| where("tags.name" => args[0]).includes(:tags) }
 
+  #
+  #
+  # Search
+  #
+  #
+
+  #
+  # Search Associations
+  #
+
+  search_association :services
+
+  #
+  # Search Attributes
+  #
+
+  search_attribute :name,
+                   type: :string
+
+  #
+  # Instance Methods
+  #
+
   # Returns whether 'host.updated.<attr>' {#notes note} is {Mdm::Note#data locked}.
   #
   # @return [true] if Mdm::Note with 'host.updated.<attr>' as {Mdm::Note#name} exists and data[:locked] is `true`.


diff --git a/app/models/mdm/service.rb b/app/models/mdm/service.rb
index 6bccb67..67193da 100755
--- a/app/models/mdm/service.rb
+++ b/app/models/mdm/service.rb
@@ -1,5 +1,7 @@
 # A service, such as an ssh server or web server, running on a {#host}.
 class Mdm::Service < ActiveRecord::Base
+  include Metasploit::Model::Search
+
   #
   # CONSTANTS
   #
@@ -175,6 +177,13 @@ class Mdm::Service < ActiveRecord::Base
   }
 
   #
+  # Search Attributes
+  #
+
+  search_attribute :name,
+                   type: :string
+
+  #
   # Validations
   #
   validates :port,
```

Now to test

```sh
(feature/MSP-10016/metasploit-data-models-search) > rspec spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb


MetasploitDataModels::Search::Visitor::Includes
  #visit
    with Metasploit::Model::Search::Group::Union
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with #children
        should return Array of all child visits
        should visit each child
    with Metasploit::Model::Search::Operation::Date
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator
    with Metasploit::Model::Search::Group::Intersection
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with #children
        should return Array of all child visits
        should visit each child
    with Metasploit::Model::Search::Operation::Integer
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator
    with Metasploit::Model::Search::Operator::Attribute
      should == []
    with Metasploit::Model::Search::Operator::Association
      should include association
    with Metasploit::Model::Search::Operation::Set::String
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator
    with Metasploit::Model::Search::Operation::Set::Integer
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator
    with Metasploit::Model::Search::Query#tree
      Metasploit::Model::Search::Query#klass
        with Mdm::Host
          with services.name
            should include :services
          with name
            should be empty
    with Metasploit::Model::Search::Operation::Null
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator
    with Metasploit::Model::Search::Operation::Union
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with #children
        should return Array of all child visits
        should visit each child
    with Metasploit::Model::Search::Operation::String
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator
    with Metasploit::Model::Search::Operation::Boolean
      it should behave like MetasploitDataModels::Search::Visitor::Includes#visit with Metasploit::Model::Search::Operation::Base
        should return operator visit
        should visit operator

Finished in 0.08222 seconds
24 examples, 0 failures
```

The rest of the specs require manual changes... so I'll just gloss over those, you can see the
[commit](https://github.com/rapid7/metasploit_data_models/commit/f0ace106f10252b4516d01ce35d591d3172a1d29). Finally,
check all the changes.

```sh
(feature/MSP-10016/metasploit-data-models-search)> git diff --cached --name-status master
A     spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/joins_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/method_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/relation_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/where_spec.rb
A     spec/dummy/config/initializers/active_record_migrations.rb
M     spec/factories/mdm/services.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_children.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/includes/visit/with_metasploit_model_search_operation_base.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_opeator_deprecated_platform.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/relation/visit/matching_record/with_metasploit_model_search_operator_deprecated_authority.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_equality.rb
A     spec/support/shared/examples/metasploit_data_models/search/visitor/where/visit/with_metasploit_model_search_group_base.rb
```

Unfortunately, when I opened the pull request, github showed redundant changes in the Changed Files tab.  This was
because `feature/exploit` didn't have the current `master` as a parent, so merge it.

```sh
(feature/MSP-10016/metasploit-data-models-search)> git merge master
Auto-merging lib/metasploit_data_models/version.rb
CONFLICT (content): Merge conflict in lib/metasploit_data_models/version.rb
Auto-merging app/models/mdm/service.rb
CONFLICT (content): Merge conflict in app/models/mdm/service.rb
Recorded preimage for 'app/models/mdm/service.rb'
Recorded preimage for 'lib/metasploit_data_models/version.rb'
Automatic merge failed; fix conflicts and then commit the result.
```

The conflicts turned out to be trivial, so those are fixed, and now the diff only shows the expected changes, so `push`
to github and check again and it worked.  Egypt pointed out that I can reproduce the Files Changed on github using

```sh
(feature/MSP-10016/metasploit-data-models-search)> git diff --name-status master...
```

Note the use of 3 '.' instead of 2.  The '..' (2 dots):

       git diff [--options] <commit> <commit> [--] [<path>...]
           This is to view the changes between two arbitrary <commit>.

       git diff [--options] <commit>..<commit> [--] [<path>...]
           This is synonymous to the previous form. If <commit> on one side is omitted, it will have the same effect as using
           HEAD instead.  

While '...' (3 dots):

   git diff [--options] <commit>...<commit> [--] [<path>...]
           This form is to view the changes on the branch containing and up to the second <commit>, starting at a common
           ancestor of both <commit>. "git diff A...B" is equivalent to "git diff $(git-merge-base A B) B". You can omit any
           one of <commit>, which has the same effect as using HEAD instead.

As you'll note in the docs, it's equivalent to `git-merge-base`, which I just learned about now, so obviously when
trying to check what the merge would look like we should use '...' because it uses the point from which a merge will be
calculated.

The final [pull request](https://github.com/rapid7/metasploit_data_models/pull/59)
