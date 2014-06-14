---
categories: git
layout: post
title: "Preserving git history when backporting features"
---

I needed to backport the `MetasploitDataModels::Search` feature from the new module cache so the team can use it on the search in the next release, which won't contain the new module cache.  Everything I could find on Google could get the file, from a branch with `git checkout <branch> <file>`, but that meant the history gets lost, from `<branch>`.  That's when I realized, that I don't care about preserving the history from `master`, since there won't be any changes relative to `master`, except those from the branch whose history I'm preserving, so I could use `git checkout <branch> <file>` trick, but use `master` for `<branch>` and layer it on top of the a branch off the feature branch to remove all code that I didn't want to backport.


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

This is everything that’s changed, but I don't want to restore all of `master`, I want preserve the feature to be backported, `lib/metasploit_data_models/search` and its accompanying specs.  First I’ll attempt just restoring all of `master`.  This will restore files to their unmodified state from `master`, without deleting files that `feature/MSP-10016/metasploit-data-models-search` had added in its history.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git checkout master -- .
(feature/MSP-10016/metasploit-data-models-search) > git status
```

So, I have a bunch of `'modified'` and `'new file'` statuses.  The `‘new file’` status is from files that exist in `master`, but were deleted in `feature/MSP-10016/metasploit-data-models-search` and are now being restored.  The `git checkout master — .` added all the files to the index, so I now need to use `git diff --cached` as `--cached` will diff against the index..

```sh
(feature/MSP-10016/metasploit-data-models-search) > git diff --cached --name-status master
```

There are added models under `app/models/mdm` that need to be removed because they're from the new module cache, but I want to keep `app/models/metasploit_data_models`, so I'll delete `app/models/mdm` added files:

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
A     db/*
A     docs/*
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
...
A     spec/app/models/metasploit_data_models/search/visitor/attribute_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/joins_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/method_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/relation_spec.rb
A     spec/app/models/metasploit_data_models/search/visitor/where_spec.rb
A     spec/db/seeds_spec.rb
A     spec/dummy/config/initializers/active_record_migrations.rb
A     spec/factories/mdm/api_keys.rb
...
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

I'll skip `config/locales.en.yml` for now as I may need to do line edits there, so the next directory to clean up is `db`.  Any new migrations or seeds can be ignored so I can just do the selective delete again.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master db`
rm 'db/migrate/*
rm 'db/seeds.rb'
```

I also know that the additions to the `docs` directory are unneeded since they document upgrade steps for the module cache.

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

`lib/metasploit_data_models/attempt.rb` is common code between `Mdm::ExploitAttempt` and `Mdm::VulnAttempt`, but I'm not taking that refactor, so it can be deleted.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm lib/metasploit_data_models/attempt.rb
rm 'lib/metasploit_data_models/attempt.rb'
```

`lib/metasploit_data_models/batch*` is part of the batching system that disables costly per record uniqueness validations for the module cache, so it's out.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm -r lib/metasploit_data_models/batch*
rm 'lib/metasploit_data_models/batch.rb'
rm 'lib/metasploit_data_models/batch/descendant.rb'
rm 'lib/metasploit_data_models/batch/root.rb'
```

The ERD system has been moved to `metasploit-erd`, but I don't want that part of this backport, so I'll remove `lib/metasploit_data_models/entity_relationship_diagram*`.

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

`NullProgressBar` is part progress bar system for `Mdm::Module::Path`, which I already removed, so `lib/metasploit_data_models/null_progress_bar.rb` can be removed too.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm lib/metasploit_data_models/null_progress_bar.rb
rm 'lib/metasploit_data_models/null_progress_bar.rb'
```

I'll obvious want to keep `lib/metasploit_data_models/search*`, so that just leaves `lib/metasploit_data_models/unique_task_joins.rb`.  I don't need it because it's part of migration refactors I already threw away.

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

So, now `app` and `lib` are good, but that means spec needs to be cleaned.  I could just iteratively run the specs until they pass, but `rspec` won't run with undefined classes in the `describe` block, so let's mirror the deletes I did in `app` and `lib` in `spec`:

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/app/models/mdm`
rm 'spec/app/models/mdm/*'
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/app/models/mdm`
rm 'spec/db/seeds_spec.rb'
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/factories/mdm`
rm 'spec/factories/mdm/*'
```

I can safely delete `spec/lib/metasploit_data_models` because the there are no specs for `lib/metasploit_data_models` code I want to preserve because they are just namespace `Module`s.

```sh
(feature/MSP-10016/metasploit-data-models-search) > git rm `git diff --cached --name-only master spec/lib/metasploit_data_models`
rm 'spec/lib/metasploit_data_models/base64_serializer_spec.rb'
rm 'spec/lib/metasploit_data_models/batch/descendant_spec.rb'
rm 'spec/lib/metasploit_data_models/batch/root_spec.rb'
rm 'spec/lib/metasploit_data_models/batch_spec.rb'
```

Thankfully, most of the shared examples have names that match the classes for which they are used, so we can delete shared examples for classes I didn't backport

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

`config/locals/en.yml` and `spec/dummy/config/initializers/active_record_migrations.rb` might not be needed, but I'm not sure, so let's test.

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake db:drop db:create db:migrate
(feature/MSP-10016/metasploit-data-models-search) > rake spec
...
/Users/luke.imhoff/git/rapid7/metasploit_data_models/app/models/metasploit_data_models/search/visitor/attribute.rb:3:in `<class:Attribute>': uninitialized constant MetasploitDataModels::Search::Visitor::Attribute::Metasploit (NameError)
...
```

Whoops, I lost `metasploit-model` because I replaced the the `gemspec` and `Gemfile` with the one from `master`, which doesn't know about `metasploit-model`.  So, I'll add `metasploit-model` back to the `gemspec`.

```ruby
  s.add_runtime_dependency 'metasploit-model', '>= 0.24.1.pre.semantic.pre.versioning.pre.2.pre.0', '< 0.25'
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

```ruby
require 'metasploit/model
```

Test again

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake spec
...
NameError: uninitialized constant Mdm::Module::Instance
...
```

Whoops, looks like the tests for `MetasploitDataModels::Search::Visitor::*` were written against the actual, new module cache classes instead of dummy classes, which is great for behavior testing on the module caching branches, but does us no good now where I don't yet have classes that can be searched.  Thankfully, from the spec comments I know what I need to supply to use a class I do have:

```ruby
        Metasploit::Model::Search::Operator::Attribute.new(
            # needs to be a real column so look up on AREL table works
            :attribute => :module_type,
            # needs to be a real class so Class#arel_table works
            :klass => Mdm::Module::Instance
        )
```

Fortunately, as part of the search usage, I know the team will want to eventually make `Mdm::Host` and `Mdm::Service` searchable, so I'll use `Mdm::Host` in place of `Mdm::Module::Instance` and `Mdm::Service` for any associations for `Mdm::Module::Instance` to test the join and include logic.

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
5 examples, 0 failures
```

Ok, so that works.  Now, let's do the same for `spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb` and rerun all the specs.

```sh
(feature/MSP-10016/metasploit-data-models-search) > rake spec
...
NoMethodError: undefined method `search_operator_by_name' for #<Class:0x007f86582a8c68>
...
```

Ok, so I need to define the search operators using the search DSL on `Mdm::Host` and `Mdm::Service`

```ruby
class Mdm::Host
  # ...
  include Metasploit::Model::Search
  # ...
  
  #
  #
  # Search
  #
  #

  #
  # Search Associations
  #

  search_association :services

  #
  # Search Attributes
  #

  search_attribute :name,
                   type: :string
  
  # ...
end
```

```ruby
class Mdm::Service
  # ...
  include Metasploit::Model::Search
  # ...
  
  #
  # Search Attributes
  #

  search_attribute :name,
                   type: :string

  # ...
end
```

Now to test

```sh
(feature/MSP-10016/metasploit-data-models-search) > rspec spec/app/models/metasploit_data_models/search/visitor/includes_spec.rb
24 examples, 0 failures
```

The rest of the specs require manual changes... so I'll just gloss over those, you can see the [commit](https://github.com/rapid7/metasploit_data_models/commit/f0ace106f10252b4516d01ce35d591d3172a1d29). Finally, check all the changes.

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

Unfortunately, when I opened the pull request, github showed redundant changes in the Changed Files tab.  This was because `feature/exploit` (the parent of `feature/MSP-10016/metasploit-data-models-search)` didn't have the current `HEAD` of `master` as a parent, so merge it.

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

The conflicts turned out to be trivial, so those are fixed, and now the diff only shows the expected changes, so `push` to github and check again and it worked.  [Egypt](https://github.com/jlee-r7) pointed out that I can reproduce the Files Changed on github using

```sh
(feature/MSP-10016/metasploit-data-models-search)> git diff --name-status master...
```

Note the use of 3 '.' instead of 2.  From `git diff --help`. For '..' (2 dots):

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

As you'll note in the docs, `...` is equivalent to `git-merge-base`, which I just learned about now, so obviously when trying to check what the merge would look like we should use `...` because it uses the point from which a merge will be calculated.

The final [pull request](https://github.com/rapid7/metasploit_data_models/pull/59)
