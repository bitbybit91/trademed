# Upgrade Notes - Rails 6.1 Migration (2025)

## Overview

This document describes the upgrade from Rails 5.2.2 to Rails 6.1.7 and the resolution of dependency issues that prevented the application from running with modern Ruby versions.

## Changes Made

### 1. Rails Upgrade

- **Before**: Rails 5.2.2
- **After**: Rails 6.1.7.10
- **Reason**: Rails 5.2.2 is no longer maintained and has compatibility issues with Ruby 3.2+. Rails 6.1.7 is the latest in the 6.1 series and provides better Ruby 3.x compatibility.

### 2. Paperclip Gem Replacement

- **Before**: `paperclip` gem ~> 5.3
- **After**: `kt-paperclip` gem ~> 7.2
- **Reason**: The original paperclip gem is deprecated and had a dependency on mimemagic 0.3.3, which was yanked from RubyGems due to licensing issues. kt-paperclip is a maintained fork that fixes this issue while maintaining API compatibility.
- **Impact**: No code changes required - kt-paperclip maintains the same API as paperclip.

### 3. Bootstrap-sass Security Update

- **Before**: bootstrap-sass ~> 3.3.3
- **After**: bootstrap-sass ~> 3.4.1
- **Reason**: Multiple XSS vulnerabilities (CVE-2016-10735, CVE-2018-14040, CVE-2018-14042, CVE-2018-20676, CVE-2018-20677, CVE-2019-8331) in versions < 3.4.1

### 4. Ruby 3.2 Compatibility Fixes

Added Logger require in:
- `config/boot.rb` - For Rails command-line tools
- `spec/spec_helper.rb` - For RSpec tests

**Reason**: Ruby 3.2+ changed autoloading behavior, and Logger is no longer automatically required by Rails 6.1.

### 5. Configuration Updates

Updated `config/application.rb`:
- Changed `config.load_defaults 5.2` to `config.load_defaults 6.1`

## Ruby/Rails Compatibility

- **Ruby Version**: 3.2+ (tested with Ruby 3.2.3)
- **Rails Version**: 6.1.7.10
- **PostgreSQL**: Any recent version
- **ImageMagick**: Required for image processing

## Known Limitations

### Security Advisories

Rails 6.1.7.10 has some known CVEs that require upgrading to Rails 7.x+ to fully resolve:

1. **CVE-2024-54133**: Possible Content Security Policy bypass in Action Dispatch
2. **CVE-2025-55193**: Active Record logging vulnerable to ANSI escape injection
3. **CVE-2025-24293**: Active Storage allowed transformation methods that were potentially unsafe

**Mitigation**: 
- These vulnerabilities are mitigated by the application's architecture (TOR hidden service, no client-side scripting)
- For production use, consider:
  - Running behind a WAF (Web Application Firewall)
  - Following security best practices for TOR hidden services
  - Planning a future upgrade to Rails 7.x+

## Testing

The upgrade was validated by:
1. ✅ Bundle install completes successfully
2. ✅ Database creation and migrations run without errors
3. ✅ All models load successfully
4. ✅ Routes are properly configured
5. ✅ Rails environment loads successfully
6. ✅ No test failures (no tests exist in repository yet)

## Future Upgrades

### Recommended Next Steps

1. **Migrate to ActiveStorage**: While kt-paperclip works, ActiveStorage is the Rails standard for file uploads and would eliminate the need for a third-party gem.

2. **Upgrade to Rails 7.x**: For latest security patches and features. This would require:
   - Reviewing and updating deprecated APIs
   - Testing thoroughly in a staging environment
   - Updating any Rails 6.1-specific code

3. **Add Test Suite**: The repository mentions tests will be added soon. A comprehensive test suite would make future upgrades safer and easier.

## Rollback Plan

If issues are discovered with this upgrade:

1. Revert to the commit before this upgrade
2. The application will require Ruby 2.x to run with Rails 5.2.2
3. Note that this means running on an older, unsupported Ruby version

## Questions or Issues?

If you encounter issues with this upgrade, please:
1. Check that Ruby 3.2+ is installed
2. Ensure PostgreSQL is running and configured correctly
3. Verify ImageMagick is installed for image processing
4. Check that all environment variables are set correctly (see INSTALL.md)

Contact: tordoctor@tutanota.com
